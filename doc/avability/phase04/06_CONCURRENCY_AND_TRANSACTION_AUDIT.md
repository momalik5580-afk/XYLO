# Phase 4 — Concurrency & Transaction Audit (GBA / Allotment)

**Method:** for every write path, record (a) whether a transaction exists, (b) whether rows are locked, (c) whether a version/condition predicate is applied, (d) whether multi-step writes can partially commit, (e) tenant scoping of each statement.

---

## 1. Summary Table

| # | Write path | Txn? | Row lock | Version predicate | Partial-commit risk | hotel_id on every stmt |
|---|---|---|---|---|---|---|
| C-1 | `CreateGroupBlockHandler` | YES (`group-block.repository.ts:103`) | none | **no** (`version:{increment:1}` `:134`) | low (single tx) | yes |
| C-2 | `CreateGroupPickupHandler` | **NO — 4 separate writes** | none | no | **HIGH** (reservation committed, counters not) | yes |
| C-3 | `CreateAllotmentVoucherHandler` | **NO** (`allotment.repository.ts:109-231`) | none | no | **HIGH** (contract upsert vs quota rows vs voucher rows) | yes |
| C-4 | `ConsumeAllotmentVoucherHandler` | **NO** | none | no | medium (fabricated reservation id) | yes |
| C-5 | `SetAllotmentQuotaHandler` | **NO — double write** (`:32` save, `:35` `upsertQuota`) | none | no | **HIGH** (quota written once, rollup recomputed separately) | yes |
| C-6 | Apply/Lift stop-sale | **NO — double write** (aggregate save + `updateQuotaStopSaleFlags`) | none | no | **HIGH** | interpolated SQL with escaped literals (`allotment.repository.ts:246-261`) |
| C-7 | `ReleaseBlockAllocationHandler` / `ReleaseAllotmentAllocationHandler` | block: yes (repo save); allotment: **no** | none | no | medium | yes |
| C-8 | `RemoveRoomCategoryHandler` | **NO** — check (`:41`) then `DELETE` (`:51`) | none | no | TOCTOU: pickup can be created between check and delete | yes |
| C-9 | `RemoveShoulderDaysHandler` | **NO** — 3 statements (`:79,:88,:100`) | none | no | **HIGH** (allocations deleted, block dates updated, counters not recomputed) | yes |
| C-10 | `AddShoulderDaysHandler` / `AddRoomCategoryHandler` | **NO** — INSERT then UPDATE block | none | no | **HIGH** | yes |
| C-11 | `DeleteGroupBlockHandler` | cancel+save in tx, then separate `delete()` | none | no | cancel persisted, delete fails → block `CANCELLED` but not deleted | yes |
| C-12 | `prisma-reservation-association.adapter` (guest → reservation → folio → routing) | **NO** | n/a | no | **HIGH** (guest orphan, reservation without folio) | yes in WHERE; `UPDATE … linkReservation` **omits hotel_id** |
| C-13 | `CancelAllotmentPickupHandler` | **NO** — SELECT `:31` → aggregate → `UPDATE allotment_pickups` `:90` → `UPDATE reservations` `:89` | none | no | **HIGH** | **NO on the two UPDATEs** (`:66,89-94`) |
| C-14 | `post-master-charge` / `record-payment` / `reconcile-room-tax` | repo/adapter-specific (folio adapter) | — | — | — | — |
| C-15 | FO `check-out` pickup status writes | **NO** (independent statements inside try/catch) | none | no | silent failure | see `check-out.handler.ts:390-392,479-481` |
| C-16 | `AllotmentRepository.save()` quota upsert | **NO** — sequential raw upserts (`:112,:159,:193,:221`) | none | no | **HIGH** (per-row failure leaves mixed state) | yes |

---

## 2. The Read-Modify-Write Pattern (core hazard)

Every aggregate-mutating command follows:

```
repo.findById(id, hotelId)     // 3-4 SELECTs, no FOR UPDATE
   …aggregate in-memory mutate…
repo.save(aggregate)           // unconditional upsert of counters from memory
```

with **no lock and no `WHERE version = ?` predicate**:

```ts
// group-block.repository.ts:124-136
update: {
  contracted_nights: block.contractedNights,
  picked_nights:     block.pickedNights,
  released_nights:   block.releasedNights,
  version:           { increment: 1 },   // unconditional
  updated_at:        new Date(),
}
```
```ts
// group-block.repository.ts:162-170 (per allocation row)
update: {
  contracted_qty: alloc.contractedQty,
  current_held_qty: alloc.currentHeldQty,
  picked_qty:     alloc.pickedQty,
  released_qty:   alloc.releasedQty,
  version:        { increment: 1 },      // unconditional
}
```
Same in `allotment.repository.ts:153` (`version: { increment: 1 }` on the contract) and the quota upsert `:226` (writes `picked`/`released` from memory).

**Consequence:** two concurrent pickups on the same block each load `picked_qty = k`, each compute `k+1`, and the last `save()` wins → **one pickup increment is lost**, while `group_pickups` contains two rows (each `save()` upserts its own pickup row). Counters then under-report consumption relative to pickup rows.

**Spec contrast (TARGET):** `gba-domain-spec.md:1215-1227` requires `UPDATE … SET …, version = version + 1 WHERE id = @blockId AND version = @expected` and `:1557-1562` recommends `SELECT … FOR UPDATE` under contention. Neither is implemented (B-5).

---

## 3. Non-Atomic Multi-Step Paths (detailed)

### 3.1 Group pickup (C-2) — the highest-exposure path

```
create-group-pickup.handler.ts
  :44  reservationAssoc.createPickupReservation()   ← commits (no txn)
        adapter :82  guest SELECT
        adapter :91  guest INSERT          (if new)
        adapter :100 reservation INSERT    ← committed
        adapter :127 folio INSERT
        adapter :131+ routing INSERT(s)
  :58  pickup.linkReservation(reservationId)        ← memory
  :65  folioPort.getMasterFolioId / postFolioCharge ← separate write
  :80  repo.save(block)                             ← own $transaction
  :81  eventBus.publishFromAggregate                ← outbox write
```

Failure matrix:

| Fails at | Persisted state | Invariant broken |
|---|---|---|
| adapter `:100` | guest row possibly created (`:91`) | orphan guest |
| adapter folio `:127` | reservation exists, no folio | reservation without folio |
| handler `:65` charge | reservation + counters OK, no charge | billing drift (logged `warn` at `:71`) |
| handler `:80` save | **reservation exists with `group_block_id`, `picked_qty` unchanged** | over-sell: Availability counts the reservation (A6) **and** the unpicked block remainder (A1) → double count until manual fix |
| handler `:81` publish | state committed, event lost | outbox/eventual consumers never notified (moot today — see C-17) |

No compensating transaction or saga exists.

### 3.2 Allotment quota double write (C-5)

```
set-allotment-quota.handler.ts
  :32  repo.save(allotment)      → allotment.repository.ts:221-230
       upsert … DO UPDATE SET picked, released, stop_sale_active, net_rate   (quota NOT written)
  :35  repo.upsertQuota(...)     → allotment.repository.ts:278-283
       INSERT … DO UPDATE SET quota = EXCLUDED.quota, version + 1
       :284-291 recompute total_daily_quota from SUM(quota) then UPDATE contract
```

- Two independent statements + a contract rollup recomputation, **no transaction**.
- If `:35` fails after `:32`, DB holds old `quota` with memory-written `picked` and an un-recomputed `total_daily_quota`.
- If the caller used `batchSetQuotas` (`:294-334`) instead, quota **is** written (`:327`) — so correctness depends on which code path invoked the repository.

### 3.3 Stop sale (C-6)

```
apply/lift-allotment-stop-sale.handler
  → aggregate mutates quota.stopSaleActive in memory (allotment.aggregate.ts:355-367 / :382-390)
  → repo.save() writes stop_sale_active via quota upsert (allotment.repository.ts:226)
  → updateQuotaStopSaleFlags() issues a SECOND UPDATE (allotment.repository.ts:259-262)
        UPDATE allotment_daily_quotas SET stop_sale_active = CASE … ELSE stop_sale_active END,
               version = version + 1 … WHERE allotment_id = '…' AND hotel_id = '…'
```

The second statement builds SQL by **string interpolation of escaped literals** (`:246-261` — `esc()` only doubles single quotes; no bound parameters), unlike the parameterized form used elsewhere (`$executeRawUnsafe(..., args)`).

### 3.4 Shoulder-day removal (C-9)

```
remove-shoulder-days.handler.ts
  :79  DELETE allocations WHERE stay_date < originalStart   (pre)
  :88  DELETE allocations WHERE stay_date > originalEnd     (post)
  :100 UPDATE group_blocks SET arrival_date, departure_date, shoulder_days_* = 0
```

- No transaction → allocations deleted while block dates unchanged (or vice versa).
- **No pickup check** (contrast `remove-room-category.handler.ts:41-48`) → pickups can survive with their allocation rows deleted → `findById` then loads a pickup whose nights no longer resolve (`DailyAllocationNotFound` on later ops).
- **No `recalculateNights()`** → `contracted_nights`/`picked_nights` remain stale relative to the surviving rows (aggregate was loaded at `:27` but never re-saved).

### 3.5 Room-category removal TOCTOU (C-8)

```
remove-room-category.handler.ts
  :41  SELECT COUNT(*) FROM group_pickups WHERE … status != 'CANCELLED'
  :51  DELETE FROM group_block_daily_allocations WHERE …
```
Between `:41` and `:51`, `CreateGroupPickupHandler` can commit a pickup (C-2 commits its own reservation independently) → pickup rows remain while allocations are hard-deleted.

### 3.6 Reservation association adapter (C-12)

`prisma-reservation-association.adapter.ts` runs guest find/create → reservation insert → folio insert → routing inserts as **separate `$queryRawUnsafe`/`$executeRawUnsafe` calls with no `$transaction`**. Additionally the `linkReservation` UPDATE omits the `hotel_id` predicate (statement region documented in `01_FORENSIC_AUDIT.md` evidence index), as does `cancel-allotment-pickup`'s `UPDATE reservations` (`:89-94`).

---

## 4. Tenant-Isolation Exceptions in Write Statements

| Statement | Scoped by `hotel_id`? | Evidence |
|---|---|---|
| `UPDATE allotment_pickups … status='CANCELLED'` (cancel pickup) | **NO** | `cancel-allotment-pickup.handler.ts:89-94` |
| `UPDATE reservations …` (cancel pickup) | **NO** | `cancel-allotment-pickup.handler.ts:66` |
| `UPDATE reservations` (`linkReservation`) | **NO** | `prisma-reservation-association.adapter.ts` (link region) |
| `UPDATE allotment_daily_quotas` (stop-sale flags) | yes (literal in WHERE) | `allotment.repository.ts:261` |
| `UPDATE allotment_contracts` (rollup) | yes | `allotment.repository.ts:290,332` |
| Repository `findFirst`/`updateMany` reads/writes | yes | `group-block.repository.ts:23,30,36,206`; `allotment.repository.ts:24,234` |
| Handlers' hotel id resolution | fallback chain to `'default'` | e.g. `create-group-block.handler.ts:26`, `set-allotment-quota.handler.ts:22`, `consume-allotment-voucher.handler.ts:15` |

Combined with the context-override behavior (`tenant.interceptor.ts:19-20` — mismatched `x-property-id` overrides the user's hotel; `resource-access.guard.ts:29` — property check disabled in development), the unscoped UPDATEs are the sharpest cross-tenant exposure in the module.

---

## 5. Event / Outbox Atomicity (C-17)

- Events are published **after** the state write (`create-group-block.handler.ts:56-57`, `create-group-pickup.handler.ts:80-81`, `set-allotment-quota.handler.ts:43`) — a crash between save and publish loses the event permanently (no outbox-in-the-same-transaction pattern is applied by these handlers).
- When an event *is* persisted, `OutboxPublisher` (`outbox-publisher.ts:36-42`) enqueues to BullMQ `events` and marks processed; `EventsConsumer` (`events.consumer.ts:46-57`) handles only `reservation.*`, so **GBA events are acknowledged and dropped** (`default:` warn at `:57`).

---

## 6. Concurrency Verdict

1. **No write path in the module uses row locks or conditional version updates**, despite `version` columns on `group_blocks`, `group_block_daily_allocations`, `allotment_contracts`, `allotment_daily_quotas`.
2. **The pickup path (C-2) can commit a reservation without committing counter changes**, producing a state Availability will double-count (reservation consumption + unpicked held remainder).
3. **Three double-write patterns** (C-5 quota, C-6 stop-sale, C-10 shoulder/category) have no transaction and no idempotency key.
4. **Four statements lack `hotel_id` scoping** (§4) in a module where header-based hotel override is accepted (`tenant.interceptor.ts:19-20`).
5. **Deadlock/serializability risk is low** (no `FOR UPDATE` anywhere, short transactions), but **lost-update risk is high** — the dominant defect class here is lost updates and partial commits, not blocking.

# Phase 4 — Legacy Dependency Map (GBA / Allotment)

**Purpose:** map every legacy artifact that the GBA domain touches or competes with, carry forward Phase 3's legacy register (L-items), and state who reads/writes what today.

---

## 1. Legacy GBA Artifacts — Status

| Artifact | Prisma model | TS writer | TS reader | Status |
|---|---|---|---|---|
| `allotment` (Int PK) | `schema.prisma:105` | none found | none found (module uses `allotment_contracts`) | **orphaned legacy** (schema + migration `20260920` only) |
| `allotment_pickup` (singular) | `:137` | none | none (module uses `allotment_pickups`, undeclared) | **orphaned legacy** |
| `allotment_room_types` | `:152` | none | none | **orphaned legacy** |
| `allotment_ledger` | `:16068` | none | none | **orphaned legacy** |
| `block_pickup` | `:991` | **none** (`rg 'INSERT INTO block_pickup|UPDATE block_pickup'` → 0) | none | **orphaned legacy** |
| `block_rates` / `block_room_type` | `:1012` / `:1030` | none | none | **orphaned legacy** |
| `reservation_block` | `:9127` | **none** (`rg 'INSERT INTO reservation_block'` → 0) | none found | **orphaned legacy** (extended by migration `20260920`) |
| `reservation_groups` | `:9100` | none | none | **schema-only** (created by `20260920`) |
| `room_blocks` | `:10288` | separate physical-rooms domain | separate | **different domain** — do not conflate |
| `orms_pickup` | `:7292` | **none** | none | **orphaned legacy** |
| `datamart_block_stats` | `:3102` | datamart pipeline | reports | **reporting only** |
| `availability` counters | `:433` | `inventory.domain-service.ts:164,181,227,256,275` | `:74,123,145,220,247` | **LIVE legacy writer** (see §3) |
| `reservations.block_code` | `:9579` | `reservation-persistence.domain-service.ts:72,81` (when supplied) | `check-in-workflow.service.ts:120` | **LIVE legacy column** |

**Search evidence:** `rg -n 'INSERT INTO block_pickup|UPDATE block_pickup|INSERT INTO orms_pickup|INSERT INTO reservation_block' -g '*.ts' apps/api/src` → **no matches**.

---

## 2. Phase 3 Legacy Register — GBA-relevant Items (carried forward)

From `docs/enterprise/availability-phase3-reservation-lifecycle-forensic-audit.md:761-773`:

| ID | Item | Phase 3 verdict | Phase 4 status after this audit |
|---|---|---|---|
| **L-13** | `group_pickups` / `allotment_pickups` writes at FO checkout — `check-out.handler.ts` (Phase 3 recorded `:334,421`; current file records them at **`:390-392` and `:479-481`**) | **KEEP until Phase 4** | **Re-verified:** writes exist, status-only, best-effort in `try/catch`, **no counter changes**, `allotment_pickups` targets an undeclared table. *Phase 4 audit does not change this — decision carried to `08_OPEN_DECISIONS.md` (D-4).* |
| L-10 | `room_inventory` read by Activities matrix, no writer | KEEP read-only or REMOVE (Phase 6) | unchanged by this audit |
| L-11 | Channel mirror `channel_availability*` never pushes into `availability` | KEEP (Phase 8) | unchanged; also: no GBA → channel path exists |
| L-12 | Restriction/sell-limit writers in Activities | KEEP — Phase 1 authority | unchanged |
| L-14 | `ReservationConsumptionAdapter` population-awareness gap | MISSING | **compounded:** GBA pickup reservations are inserted by raw SQL and never registered with the assertion engine (`04_AVAILABILITY_INTEGRATION_MATRIX.md` §1, A5) |
| L-16 | Assertion engine built, unwired | MISSING (wiring) | unchanged by this audit (Phase 3 scope) |
| L-18 | Outbox consumers touching availability: none | MISSING if async chosen | **extended finding:** outbox consumer exists (`events.consumer.ts`) but drops **all** GBA events (`:57`) |

---

## 3. The Legacy Counter Writer (still live)

`apps/api/src/modules/rates-inventory/domain/services/inventory.domain-service.ts`:

| Function | Line | Called by | Writes |
|---|---|---|---|
| `reserve(tx, …)` | :164 (INSERT), :181 (UPDATE) | `crs-engine.service.ts:391,531,542` | `availability.reserved`, `available` |
| `release(tx, …)` | :227, :256 | `crs-engine.service.ts:533,535,539,589,595` | `availability.reserved`, `available` |
| `assertAvailability` | :123 | `crs-engine.service.ts:345` | read `available` |
| `checkAvailability` | :74, :145 | `crs-engine.service.ts:97` | read |
| `isAvailable` | :123 | `front-office/upgrade-room.handler.ts:69` | read |
| `blockAvailability` | :208 | **no caller** | legacy block counter |
| `consumePickup` | :237 | **no caller** | legacy block pickup counter |
| `releaseUnsold` | :266 | **no caller** | legacy wash |

`CrsEngineService` is consumed outside rates-inventory by `crs-front-desk-integration.service.ts:15`, `availability/…/unresolved-restriction.adapter.ts:8`, and `reservations/…/reservation.repository.ts:83` (used at `:625` for `modifyReservation`).

**Relationship to GBA:** none. The three legacy block/pickup/wash functions (`:208,:237,:266`) — the *legacy equivalents of what Phase 4 would need* — are dead. The new GBA module neither reads nor writes `availability`.

---

## 4. Legacy vs New — Reservation Linkage

| Mechanism | Written by | Read by | World |
|---|---|---|---|
| `reservations.block_code` (String) | `reservation-persistence.domain-service.ts:72,81` | `front-office/check-in/…/check-in-workflow.service.ts:120` (`blockCode: row.block_code ?? null`) | legacy |
| `reservations.group_block_id` (String → `group_blocks.id`) | `prisma-reservation-association.adapter.ts:105,112` | availability A1 path via `group_pickups`/allocations; FO | **new (authoritative for pickup)** |
| `reservations.allotment_id` (Int → legacy `allotment`) | adapter writes **`NULL`** for new-world allotment pickups | none found | legacy type, unusable with `allotment_contracts` (String id) — `03_DATA_MODEL_AUDIT.md` D-1 |
| `reservations.allotment_voucher_id` | adapter stores **voucher code** (not FK) | none found | hybrid |
| `reservations.pickup_type` (`'group_pickup'`, `'GROUP_BLOCK'`) | adapter `:105,111-112` | none found | new |
| `group_pickups.reservation_id` | aggregate `linkReservation` after adapter returns id | `remove-room-category` (`:41`), repo loads | new |
| `allotment_vouchers.reservation_id` | `consumeVoucher` — may be **fabricated** (`consume-allotment-voucher.handler.ts:18`) | `allotment.controller.ts:290` join | new, unreliable |
| DTO `block_code` accepted on create | `create-reservation.dto.ts:26` — **never used by `create-reservation.handler`** | — | dead field |

**Two worlds, one table:** `reservations` simultaneously carries a legacy `block_code`, an Int `allotment_id` pointing at a legacy table, and a new `group_block_id` pointing at the new table. Nothing enforces mutual exclusivity.

---

## 5. Front Office Dependencies

| Operation | GBA interaction | Evidence |
|---|---|---|
| Checkout | status write-back only | `check-out.handler.ts:390-392,479-481` |
| Check-in | reads `block_code` into workflow context | `check-in-workflow.service.ts:120` |
| Room upgrade | legacy `inventoryDomain.isAvailable` + assertion port | `upgrade-room.handler.ts:69,34` |
| Anything else | none | module grep → only `check-out.handler.ts` |

No FO command can create/cancel pickups or advance block status; conversely, GBA commands do not call FO.

---

## 6. Dead-Code Inventory (GBA-related)

| Item | Location | Why dead |
|---|---|---|
| `IInventoryCommitmentPort` | `group-allotment/domain/ports/inventory-commitment.port.ts` | zero implementations, zero bindings |
| `ReleaseWindow.isWithinReleaseWindow` | `release-window.value-object.ts:26` | zero call sites |
| `AttritionCalculationService` | provided `group-allotment.module.ts:72,121` | never injected by any consumer |
| `AttritionPolicy` VO | `domain/value-objects/` | no evaluation path |
| `blockAvailability` / `consumePickup` / `releaseUnsold` | `inventory.domain-service.ts:208,237,266` | zero callers |
| Legacy GBA tables (§1) | `allotment`, `allotment_pickup`, `block_pickup`, `reservation_block`, `orms_pickup`, … | schema-only; no TS reads/writes |
| DTO `block_code` / `allotment_id` on create | `create-reservation.dto.ts:26-27` | accepted, never consumed by handler |
| 22 `IntegrationEvent` classes | `group-allotment.events.ts:3-396` | published → dropped by `events.consumer.ts:57` |

---

## 7. Dependency Verdict

1. **GBA does not depend on any legacy GBA table** — it has built a complete parallel world; legacy GBA tables are now schema-only (except `reservations.block_code` and the FO `check-in` read of it).
2. **GBA does not depend on, or feed, the legacy `availability` counters** — the live legacy writer (`inventory.domain-service` via CRS engine) and the new GBA module operate on disjoint state, while Availability Phase 1/2 reads GBA directly.
3. **The one live legacy→new bridge is `reservations`**: legacy columns and new columns coexist on the same table with no mutual-exclusion rule (§4).
4. **Phase 3's L-13 is re-verified with updated line numbers** and remains undecided pending Phase 4 decisions (§2).

# Phase 4 — Forensic Audit (GBA / Allotment Integration)

**Status:** Complete (audit only — no code, schema, migration, or behavior changed)
**Date:** 2026-09-30
**Scope:** Group Block & Allotment (GBA) domain as it exists in code today, its data model, and its relationship to Availability.

---

## 1. Scope and Non-Goals

**In scope (as executed):**

- Current-state discovery of the GBA domain, its database model, and its Availability touchpoints.
- Evidence collection with `file:line` citations for every conclusion.
- Identification of drift between `packages/db/schema.prisma`, migrations, and raw SQL executed at runtime.
- Identification of divergence between Availability's read path, the Activities availability matrix, and legacy counter logic.
- Production of nine deliverables in `docs/availability/phase-4/`.

**Explicitly not done (per task rules):**

- No implementation, refactoring, schema changes, migrations, backfill, API changes, or behavior changes.
- No reopening of Phase 1–3 work (`docs/enterprise/availability-phase3-*` are read as evidence only).
- No implementation planning — decisions are surfaced in `08_OPEN_DECISIONS.md`, not resolved.

**Existing reference material consulted (not duplicated):**

| Document | Role in this audit |
|---|---|
| `docs/design/gba-domain-spec.md` | TARGET-state design spec. Not ratified code. Used only to contrast against CURRENT behavior (see `05_BUSINESS_RULES_CURRENT_STATE.md`). |
| `docs/enterprise/availability-phase3-reservation-lifecycle-forensic-audit.md` | Phase 3 audit; legacy item register L-1…L-18 (esp. **L-13** `group_pickups`/`allotment_pickups` writes at FO checkout) is carried forward in `07_LEGACY_DEPENDENCY_MAP.md`. |
| `docs/enterprise/availability-phase3-*.md` | Phase 1–3 authority definitions (assertion engine, reservation consumption, snapshot calculator). |
| `docs/enterprise/reservations-roadmap-v2.md` | Exit-gate language reused for the Phase 4 gate in `09_PHASE_4_AUDIT_SUMMARY.md`. |

---

## 2. Method

1. **Schema forensics** — every GBA table read from `packages/db/schema.prisma`; every table name referenced in GBA raw SQL cross-checked against `packages/db/migrations/**` and against Prisma models.
2. **Code forensics** — all 96 files of `apps/api/src/modules/group-allotment/` read or grepped: 25 command handlers, 5 query handlers, 2 controllers, 2 repositories, 3 adapters, 4 domain ports, aggregates/value-objects/events.
3. **Integration forensics** — grep across `apps/api/src/**` for every GBA table name to find writers/readers outside the module (Front Office, Activities, Reservations, Rates-Inventory, Availability).
4. **Test inventory** — file counts and `*.spec.ts` search per module.
5. **Consumer inventory** — `apps/web/**` API-path grep for GBA routes.

**Evidence rules applied:** no conclusion without a citation; behavior is stated only from executed code (not from names, comments, or the design spec); CURRENT vs TARGET are kept in separate sections.

---

## 3. What Exists — Inventory

### 3.1 Module

`apps/api/src/modules/group-allotment/` — **96 `.ts` files, 0 test files.**

| Layer | Contents |
|---|---|
| `api/controllers/` | `group-booking.controller.ts` (`@Controller('group-bookings')`, `@PropertyScope(true)`), `allotment.controller.ts` (`@Controller('allotments')`) |
| `application/commands/` | **25 commands**: create/confirm/delete group-booking, create/delete/set-daily-allocation/release-block-allocation/add-shoulder-days/remove-shoulder-days/add-room-category/remove-room-category group-block, create-group-pickup, create-allotment, set-allotment-quota, create-allotment-voucher, consume-allotment-voucher, cancel-allotment-voucher, create-allotment-pickup, cancel-allotment-pickup, apply/lift-allotment-stop-sale, release-allotment-allocation, post-master-charge, record-payment, reconcile-room-tax |
| `application/queries/` | **5 queries**: get-group-bookings, get-group-booking-detail, get-group-daily-grid, get-allotments, get-master-folio |
| `domain/aggregates/` | `group-block.aggregate.ts`, `allotment.aggregate.ts`, group-booking aggregate |
| `domain/value-objects/` | `allocation-status.value-object.ts`, `daily-room-allocation.value-object.ts`, `release-window.value-object.ts`, `attrition-policy.value-object.ts` |
| `domain/services/` | `lifecycle.service.ts`, `attrition-calculation.service.ts` |
| `domain/ports/` | `inventory-commitment.port.ts`, `reservation-association.port.ts`, `folio.port.ts`, `billing-instruction.port.ts` |
| `domain/events/` | `group-allotment.events.ts` — **22 `IntegrationEvent` classes** |
| `infrastructure/repositories/` | `group-block.repository.ts`, `allotment.repository.ts`, group-booking repository |
| `infrastructure/adapters/` | `prisma-reservation-association.adapter.ts` (+ folio/billing adapters) |
| `permissions/` | `GROUP_BOOKING_PERMISSIONS` (VIEW/CREATE/CONFIRM/MODIFY, GROUP_BLOCK_VIEW/CREATE/MODIFY/PICKUP/RELEASE, ALLLOTMENT_VIEW/VOUCHER) |

### 3.2 Database — NEW (GBA-native) tables

| Table | Prisma model | Migration that creates it |
|---|---|---|
| `group_bookings` | `schema.prisma:17046` | **NONE** |
| `group_blocks` | `schema.prisma:17079` | **NONE** |
| `group_block_daily_allocations` | `schema.prisma:17113` | **NONE** |
| `group_pickups` | `schema.prisma:17136` | **NONE** |
| `allotment_contracts` | `schema.prisma:17161` | **NONE** |
| `allotment_daily_quotas` | `schema.prisma:17198` | **NONE** |
| `allotment_vouchers` | `schema.prisma:17221` | **NONE** |
| `allotment_stop_sales` | `schema.prisma:17252` | **NONE** |
| `group_block_analytics` / `allotment_analytics` | `schema.prisma` (analytics models) | **NONE** |

49 migrations exist under `packages/db/migrations/`. `rg -F 'group_blocks' packages/db/migrations` and `rg -F 'allotment_contracts' packages/db/migrations` both return **zero files**. The only GBA-adjacent migration, `20260920_groups_blocks_allotments/migration.sql`, creates `reservation_groups` and ALTERs legacy `reservation_block` + `allotment` — it does not create any of the tables above.

### 3.3 Database — LEGACY (parallel GBA world)

| Table | Prisma model | Evidence of use today |
|---|---|---|
| `allotment` (Int PK) | `schema.prisma:105` | extended by migration `20260920` (`contract_type`, `call_in_days`, `billing_routing_config`); `reservations.allotment_id` (Int) points here |
| `allotment_pickup` (singular) | `schema.prisma:137` | legacy pickup rows |
| `allotment_room_types` | `schema.prisma:152` | legacy quota-by-room-type |
| `allotment_ledger` | `schema.prisma:16068` | legacy ledger |
| `block_pickup` | `schema.prisma:991` | legacy block pickup |
| `block_rates` | `schema.prisma:1012`, `block_room_type` `:1030` | legacy block rate/type |
| `reservation_block` (Int PK) | `schema.prisma:9127` | extended by `20260920` |
| `reservation_groups` | `schema.prisma:9100` | created by `20260920` |
| `room_blocks` | `schema.prisma:10288` | **physical** room blocks (distinct concept) |
| `datamart_block_stats` | `schema.prisma:3102` | datamart |
| `orms_pickup` | `schema.prisma:7292` | legacy ORMS |
| `availability` (counters `alloted`/`reserved`/`available`/`oversell`/`waitlist`) | `schema.prisma:433` | written by `inventory.domain-service.ts:164,181,227,256,275` |

**No writer of `block_pickup`, `orms_pickup`, or `reservation_block` was found in TypeScript** (`rg 'INSERT INTO block_pickup|UPDATE block_pickup|INSERT INTO orms_pickup|INSERT INTO reservation_block' apps/api/src` → zero hits).

### 3.4 Database — referenced but absent everywhere

`allotment_pickups` (**plural**) is executed in raw SQL at:

- `create-allotment-pickup.handler.ts:130` (`INSERT INTO allotment_pickups`)
- `cancel-allotment-pickup.handler.ts:31,90` (SELECT / UPDATE)
- `allotment.controller.ts:290` (SELECT, joined to `reservations`)
- `front-office/.../check-out.handler.ts:392` (`UPDATE allotment_pickups SET status='CHECKED_OUT'`)

It is **not a Prisma model** (`rg -F 'allotment_pickups' packages/db/schema.prisma` → no hit; only `allotment_pickup` singular at `:137`) and **not in any migration** (`rg -l -F 'allotment_pickups' packages/db/migrations` → no file; `rg -l -F 'allotment_pickups' -g '*.sql' packages/db` → no file).

Similarly, `group_blocks.shoulder_days_before` / `shoulder_days_after` are read and written by raw SQL (`remove-shoulder-days.handler.ts:31-36,100-112`, `add-shoulder-days.handler.ts:116`) but are **absent from the `group_blocks` model** (`schema.prisma:17079-17111`) and from all migrations (`rg -l -F 'shoulder_days' packages/db/migrations` → no file).

### 3.5 HTTP surface

`group-booking.controller.ts`: `GET form-data` (:56), `GET available-rooms` (:95), `GET /` (:118), `GET :id` (:136), `GET/POST :id/master-folio*` (:143,:150,:163,:177), `POST /` (:186), `POST :id/confirm` (:194), `GET/POST :id/blocks` (:201,:208), `GET :bookingId/blocks/:blockId/grid` (:219), `PUT …/allocations` (:226), `POST …/pickups` (:236), `POST …/release` (:247), `POST …/shoulder` (:257), `POST …/shoulder/remove` (:267), `POST …/categories` (:277), `POST …/categories/remove` (:287), `DELETE :id` (:297), `DELETE :bookingId/blocks/:blockId` (:304).

`allotment.controller.ts`: `GET form-data` (:48), `GET /` (:121), `GET :id/negotiated-rates` (:138), `GET :id` (:159), `POST /` (:166), `PUT :id/quotas` (:174), `POST :id/vouchers` (:184), `POST :id/stop-sales` (:195), `POST :id/vouchers/:vid/consume` (:206), `POST :id/vouchers/:vid/cancel` (:217), `POST :id/stop-sales/:sid/lift` (:228), `POST :id/release` (:239), `POST :id/pickups` (:251), `POST :id/pickups/:pid/cancel` (:271), `GET :id/pickups` (:281), `DELETE :id` (:321).

### 3.6 Test inventory

| Scope | `*.spec.ts` |
|---|---|
| `apps/api/src/modules/group-allotment/**` | **0** (25 command handlers, 5 queries, 2 controllers, 2 repos: untested) |
| GBA-touching test files repo-wide | 4 — `availability-source.adapter.spec.ts`, `snapshot-calculator` spec(s), `availability-snapshot.service.spec.ts`, plus one Front Office fixture (`front-office/check-in/__tests__/application/fakes.ts:34` carries `block_code: null`) |
| `apps/api/test/` (e2e) | empty |

---

## 4. Evidence Index (most-cited locations)

| Topic | Citation |
|---|---|
| GBA → Availability read path | `apps/api/src/modules/availability/infrastructure/adapters/availability-source.adapter.ts:49,93` |
| Snapshot consumption formula | `apps/api/src/modules/availability/domain/policies/snapshot-calculator.ts` (`consumption = reservationConsumption + gbaRemaining + allotmentRemaining`) |
| Second, divergent availability computation | `apps/api/src/modules/activities/availability-sales.controller.ts:314-335,453-459` |
| Dead port (never implemented/bound) | `apps/api/src/modules/group-allotment/domain/ports/inventory-commitment.port.ts` |
| Dead legacy GBA functions | `apps/api/src/modules/rates-inventory/domain/services/inventory.domain-service.ts:208,237,266` |
| Pickup → reservation (raw SQL, no tx) | `apps/api/src/modules/group-allotment/infrastructure/adapters/prisma-reservation-association.adapter.ts:61-131` |
| FO checkout writes GBA pickup status | `apps/api/src/modules/front-office/application/commands/check-out/check-out.handler.ts:390-392,479-481` |
| `'default'` hotel fallback | every GBA handler, e.g. `create-group-block.handler.ts:26` |
| Outbox → BullMQ → no GBA handler | `apps/api/src/common/outbox/outbox-publisher.ts`, `apps/api/src/modules/shared/events.consumer.ts:46-57` |
| Property-context override | `apps/api/src/core/interceptors/tenant.interceptor.ts:19-20`, `apps/api/src/common/authorization/resource-access.guard.ts:29` |

---

## 5. Audit Boundary Statement

Nothing outside `docs/availability/phase-4/*.md` was created, modified, or executed against the database during this audit. All commands run were read-only (`rg`, `Get-Content`, `Get-ChildItem`).

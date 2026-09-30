# Phase 4 — Open Decisions

**Rule:** no decision is made in this audit. Each item states the question, the evidence, the options visible in the codebase, and what is blocked behind it. Owners/priorities are suggestions, not commitments.

---

## D-1 — Which GBA data world is authoritative?

**Evidence:** two complete worlds (`allotment`+`block_*` legacy vs `allotment_contracts`+`group_blocks` new) with a type-level barrier: `reservations.allotment_id` is `Int` → legacy `allotment`; `allotment_contracts.id` is `String` (`03_DATA_MODEL_AUDIT.md` D-1). Legacy tables have **no TS writers/readers** (`07_LEGACY_DEPENDENCY_MAP.md` §1).

| Option | Implication |
|---|---|
| A. Declare new world authoritative; legacy tables become Phase-11 removal candidates | matches where all runtime code points |
| B. Retain legacy tables as reporting/datamart inputs | requires documenting them as read-only |
| C. Merge | requires an `Int ↔ String` mapping plan for `reservations.allotment_id` |

**Blocks:** Phase 11 legacy removal, any migration strategy for the new tables.

---

## D-2 — How do the new GBA tables get into the database reproducibly?

**Evidence:** zero migrations create `group_bookings`/`group_blocks`/`group_block_daily_allocations`/`group_pickups`/`allotment_contracts`/`allotment_daily_quotas`/`allotment_vouchers`/`allotment_stop_sales`/analytics (`01_FORENSIC_AUDIT.md` §3.2); `allotment_pickups` and `shoulder_days_*` are referenced by raw SQL but exist in **no** migration and (for the table) in no Prisma model (`03_DATA_MODEL_AUDIT.md` §2).

| Option | Implication |
|---|---|
| A. Write a forward migration matching live state | requires first **verifying live DB state** (not done — no DB access in this audit) |
| B. Accept `db push` as the mechanism for this domain | diverges from migrate-based workflow used elsewhere (49 migrations) |
| C. Remove shadow artifacts (`allotment_pickups`, `shoulder_days_*`) from code | changes behavior of pickup + shoulder features |

**Prerequisite verification (read-only):** does `information_schema.tables` contain the new tables, `allotment_pickups`, and `group_blocks.shoulder_days_*` in the target environment?
**Blocks:** every migration-dependent plan; FO checkout write-back correctness (L-13).

---

## D-3 — Cut-off wash and rolling release: build now, later, or never?

**Evidence:** `cutoff_date` stored, never read; `release_days_before`/`is_rolling_release` stored, never evaluated; `ReleaseWindow.isWithinReleaseWindow` has 0 call sites; no scheduler exists (`05_BUSINESS_RULES_CURRENT_STATE.md` B-1/B-2; `04_…_MATRIX.md` §4). Manual release exists (`release-block-allocation`, `release-allotment-allocation`).

| Option | Implication |
|---|---|
| A. Implement scheduled wash (Temporal job exists as platform pattern) | restores spec T-1/T-2; requires defining wash semantics per `inventory_policy`/`contract_type` |
| B. Keep manual release; delete the dead `ReleaseWindow` + release fields | simplest; diverges from spec |
| C. Defer to a later phase | keeps stored-but-unenforced fields (data trap: users see cutoff dates with no effect) |

**Blocks:** availability accuracy over time (currently a block holds inventory indefinitely until manually released or status changes).

---

## D-4 — FO checkout ↔ pickup status: keep, move, or replace? (Phase 3 L-13)

**Evidence:** `check-out.handler.ts:390-392,479-481` writes `allotment_pickups`/`group_pickups` status in try/catch; no counter changes; `allotment_pickups` may not exist (D-2). Phase 3 verdict: "KEEP until Phase 4".

| Option | Implication |
|---|---|
| A. Keep as-is | status-only, silent-failure behavior persists |
| B. Move into GBA via an event consumer (`reservation.checked_out` → GBA handler) | requires D-5 (event consumption) |
| C. Replace with reservation-status-driven derivation (drop pickup status column semantics) | largest change; removes a source of divergence |

**Blocks:** closing L-13; consistent "consumed" reporting for blocks/allotments.

---

## D-5 — What consumes the 22 GBA events?

**Evidence:** all 22 `IntegrationEvent`s are published to outbox → BullMQ → `events.consumer.ts`, whose switch handles only `reservation.*` and logs `no handler` for everything else (`02_CURRENT_ARCHITECTURE.md` §4).

| Option | Implication |
|---|---|
| A. Add handlers for availability cache invalidation + analytics projection | minimal useful set |
| B. Publish externally (channels/CRS) per spec T-10 | requires target system contract |
| C. Stop emitting events until a consumer exists | removes outbox churn; loses audit trail |

**Blocks:** D-4 option B; any real-time UI updates; spec T-10.

---

## D-6 — Which "available" number is authoritative: A1 (Availability) or A3 (Activities matrix)?

**Evidence:** `availability-source.adapter.ts:49` vs `availability-sales.controller.ts:306-335,459` differ on **every** eligibility filter (`04_AVAILABILITY_INTEGRATION_MATRIX.md` §3): `deleted_at`, booking status, block-status set, `contract_type`, validity window, plus OOO and restriction handling.

| Option | Implication |
|---|---|
| A. Availability (A1) is authoritative; Activities matrix reimplemented on top of it | single rule source |
| B. Keep both, document them as different views ("sellable" vs "physical minus commitments") | must relabel UI and define semantics |
| C. Align A3's filters to A1 with a minimal SQL patch | smallest change, still two implementations |

**Blocks:** user-facing trust in availability numbers; Phase 6 inventory/rates integration scope.

---

## D-7 — Should GBA pickup validate against transient availability?

**Evidence:** pickup checks only intra-block capacity (`group-block.aggregate.ts:234-242`); no Availability/assertion port call; `IInventoryCommitmentPort` is unwired (`05_…_CURRENT_STATE.md` B-3).

| Option | Implication |
|---|---|
| A. Wire pickup to the assertion engine (call `ReservationAvailabilityPort`) | aligns with Phase 2 authority; pickup-created reservations stop bypassing balances (matrix A5) |
| B. Define block pickup as exempt (consumes block pool only) — then document that non-DEDUCT policies never touch transient inventory | matches current code; needs explicit rule |
| C. Wire only for `DEDUCT_INVENTORY` blocks | middle path |

**Blocks:** closing the assertion-engine bypass (L-14 compounding).

---

## D-8 — Concurrency model: which guarantee do we adopt?

**Evidence:** `version` columns exist on all GBA aggregates/children but are written with unconditional `{ increment: 1 }` and never used as a predicate; no `FOR UPDATE` anywhere (`06_CONCURRENCY_AND_TRANSACTION_AUDIT.md` §2). Spec T-5 requires conditional updates.

| Option | Implication |
|---|---|
| A. Conditional update (`WHERE version = ?`) + retry | matches spec; requires threading expected version through commands/APIs |
| B. `SELECT … FOR UPDATE` per aggregate at load | simpler for pick-up hot spots; holds locks across aggregate save |
| C. Accept last-write-wins | document lost-update risk; pickup counters can under-count |

**Blocks:** correctness of `picked`/`picked_qty` under concurrent pickup (the module's core invariant).

---

## D-9 — Pickup transaction boundary

**Evidence:** `create-group-pickup.handler.ts` commits reservation (adapter, no txn) → folio charge → `repo.save(block)` in three units; failure between them leaves reservation without counter increment (`06_…` C-2, §3.1).

| Option | Implication |
|---|---|
| A. One `$transaction` spanning reservation + counters + pickup row | requires adapter to accept a tx handle (currently uses `this.prisma` directly) |
| B. Outbox/saga with compensating cancel | heavier; matches event-driven style |
| C. Order changes: counters first, then reservation, with compensating counter restore | still needs compensation |

**Blocks:** eliminating double-count window; making `group_pickups`↔`reservations` linkage reliable.

---

## D-10 — Fabricated voucher reservation id

**Evidence:** `consume-allotment-voucher.handler.ts:18` — `const reservationId = command.data.reservationId || \`RES-${Date.now()}\``.

| Option | Implication |
|---|---|
| A. Require `reservationId` (reject if missing) | may break existing UI callers (`group-allotment.api.ts:303` posts body possibly empty) |
| B. Create the reservation as part of consume (like pickup does) | matches spec T-9 voucher intake |
| C. Allow null and store null | loses linkage, honest about state |

**Blocks:** trustworthy voucher↔reservation reporting.

---

## D-11 — Guest resolution by `full_name ILIKE`

**Evidence:** `prisma-reservation-association.adapter.ts:82-97` matches guests by hotel + case-insensitive full name, else creates.

| Option | Implication |
|---|---|
| A. Keep (convenience for group pickup) | same-name guests merge across reservations |
| B. Always create a new guest | duplicates |
| C. Accept explicit `guestId`/email match first | needs API change |

**Blocks:** guest-data integrity review (may already be covered by other phases — flagged for owner confirmation).

---

## D-12 — HTTP verb contract for block release

**Evidence:** frontend `api.put('/group-bookings/…/release')` (`group-allotment.api.ts:146`) vs controller `@Post('…/release')` (`group-booking.controller.ts:247`) → 405 for the block-release action.

| Option | Implication |
|---|---|
| A. Change frontend to POST | one-line fix |
| B. Add `@Put` alias on controller | keeps both callers working |

**Blocks:** nothing functional beyond the broken action itself.

---

## D-13 — Shoulder-day schema artifacts

**Evidence:** `shoulder_days_before/after` used by add/remove shoulder handlers, absent from `schema.prisma` and migrations (`03_…` §2, D-4).

| Option | Implication |
|---|---|
| A. Add columns to Prisma model + migration | makes code honest |
| B. Remove shoulder feature | removes capability (`POST …/shoulder` routes + UI) |

**Blocks:** D-2 (migration strategy must include these columns if they exist live).

---

## D-14 — Attrition: activate or remove?

**Evidence:** `AttritionCalculationService` provided but never injected; `attrition_threshold` stored (default 80); no computation path (`05_…` B-7).

| Option | Implication |
|---|---|
| A. Implement attrition report/penalty flow (spec T-7) | feature work |
| B. Remove service + threshold column from model | honest schema |

**Blocks:** nothing today; `group_blocks.attrition_threshold` is currently decorative.

---

## D-15 — Block lifecycle: how does a block become `DEFINITE` / `OPEN_FOR_PICKUP`?

**Evidence:** aggregate transition methods exist (`group-block.aggregate.ts:131 confirm`, `:138 openForPickup`, `:144 close`, all guarded by `assertTransition` at `:119`) but **no command invokes them**; `washAllocation` (`:332`) likewise. Blocks are created `DRAFT` (`schema.prisma:17085`). Availability counts only `DEFINITE`/`OPEN_FOR_PICKUP` blocks (`availability-source.adapter.ts:66`), so command-created blocks currently contribute **0** consumption to the snapshot. Allotments are `activate()`d at creation (`create-allotment.handler.ts:49`) and do count.

| Option | Implication |
|---|---|
| A. Add `ConfirmGroupBlock` / `OpenBlockForPickup` / `CloseGroupBlock` commands + routes (spec T-6) | restores intended behavior; requires permission definitions and tests |
| B. Auto-advance on a trigger (e.g., booking confirm → blocks become DEFINITE) | implicit state change; must define which trigger |
| C. Change A1 eligibility to include `DRAFT`/`TENTATIVE` | misrepresents unconfirmed commitments as held inventory |
| D. Do nothing | blocks never suppress transient inventory — the DEDUCT feature is inert |

**Blocks:** any correct reading of A1 consumption; spec T-6; interacts with D-6 (authoritative availability number).

---

## Decision Register Summary

| ID | Theme | Depends on | Suggested priority |
|---|---|---|---|
| D-2 | Schema/migration reproducibility | live DB verification | **highest** — gates everything |
| D-15 | Block lifecycle command set | D-2 | **high** — A1 is otherwise inert for blocks |
| D-8 | Concurrency guarantee | — | **high** — core invariant |
| D-9 | Pickup transaction boundary | D-8 | **high** |
| D-7 | Pickup vs transient availability | D-6 | high |
| D-6 | Authoritative availability number | — | high |
| D-1 | Authoritative data world | D-2 | medium |
| D-3 | Wash/release automation | D-2 | medium |
| D-5 | Event consumers | — | medium |
| D-4 | FO checkout (L-13) | D-5, D-2 | medium |
| D-10, D-11, D-12, D-13, D-14 | local correctness/cleanup | D-2 | lower |

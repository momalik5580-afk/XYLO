# XYLO Availability Phase 4 — Final Domain Specification

**Phase:** 4 — GBA / Allotment Integration
**Version:** 1.0 — **Status: FINAL / READY FOR IMPLEMENTATION PLAN**
**Date:** 2026-09-30
**Document type:** Domain contract (NOT an implementation plan)

**Verified state at publication:**

| Metric | Value |
|---|---|
| Business decisions | **22/22 resolved** (0 `USER DECISION REQUIRED`) |
| Confirmations | **6/6 confirmed** (D-2, D-4, D-6a, D-10, S-1, S-3 — 2026-09-30) |
| D-2 live-DB verification | **PASS** (read-only `information_schema`, 2026-09-30) |
| Target rules | **97 = 95 DECIDED · 0 RECOMMENDED · 0 PENDING USER · 2 DEFERRED** |
| Decision Resolution gate | **FULLY CLOSED** |

**Supersession statement:** this document is the authoritative domain contract for Phase 4. **It supersedes the earlier design-only `docs/design/gba-domain-spec.md` where they conflict.** Every supersession is listed in §20.2. No resolved decision is reopened, reinterpreted, or extended here.

**Source-of-truth order used (conflicts resolved upward, never silently):**

1. `10_DECISION_RESOLUTION.md` (resolved decisions)
2. `11_TARGET_BUSINESS_RULES.md` (97 target rules, TR-x)
3. `12_DECISION_STATUS.md` (status/gate)
4. Phase 4 audit `01`…`09` (evidence, findings F-x, current-state B-x/C-x/A-x/L-x)
5. `docs/design/gba-domain-spec.md` (design reference)
6. Current code / schema / live-DB evidence

**Provenance tags used throughout:**
`[DECIDED]` = resolved decision/target rule (binding) · `[DEFERRED]` = intentionally implementation-plan scope (a mechanism, not a rule) · `[SPEC-CARRIED]` = from spec/audit, not contradicted by any decision, carried with provenance (may not be presented as a "decided" rule) · `SOURCE-SILENT` = no rule exists; none invented.

---

## 1. Domain Purpose and Scope

### 1.1 What Phase 4 owns

Phase 4 defines the complete domain contract for **Group Booking & Allotment (GBA)**: the bounded context that holds contracted room commitments with groups and partners and governs how those commitments are consumed, released, and enforced — and its contracts with every neighboring context.

**Owned by this domain (Phase 4 scope):**

| Sub-domain | Owned concepts |
|---|---|
| **Group blocks** | block identity/lifecycle, daily allocations, contracted quantities, cut-off, wash, manual release, attrition, shoulder days, block pickup |
| **Allotments** | contracts, daily quotas, contract types, rolling release, stop sale, vouchers, allotment pickup, expiry |
| **Pickup (consumption)** | the canonical pickup record, guards, atomicity, counter truth, voucher authorization flow |
| **Voucher** | issuance, consumption, cancellation, expiry as an authorization artifact |
| **GBA domain events** | emission contract and mandatory consumer coverage |
| **GBA data world** | the 9 new-world tables + shadow objects (D-2 declaration), legacy boundary (D-1) |

**Owned by other contexts (Phase 4 defines the contract, not the behavior):**

| Concern | Owner |
|---|---|
| Reservation lifecycle (create/confirm/guarantee/cancel/check-in/check-out/no-show/modify) | **Reservations / Front Office** |
| Physical inventory and sellable-availability computation (authority A1) | **Availability** |
| Reservation assertion engine (Phase 2 balances) | **Availability/Reservations** (pickup must pass through it — D-7) |
| Guest identity store | **Reservations/Profiles** (GBA only applies the D-11 resolution order at pickup) |
| Financial ledger ownership of `ATTRITION_FEE` posting | **Existing financial/folio owner** (posting-in-wash is decided; ledger ownership = implementation scope, spec open question 2 `gba-domain-spec.md:1718`) `[DEFERRED]` |
| Front Office room assignment, folio windows, check-in/out execution | **Front Office** |
| External CRS/channel publication (spec T-10) | **Later integration step** (D-5: not required to close this phase) `[DEFERRED]` |

### 1.2 Explicitly out of scope (not introduced by this document)

- Phase 5+ functionality; external channel/CRS push contracts; multi-property contracts (TR-14.3: all Phase 4 rules are single-hotel).
- BEO/function-space detail (spec §15 child-entity design — no Phase 4 decision covers it).
- Analytics computation decisions (real-time vs batch — spec open question 7; backlog).
- Event sourcing, aggregate size limits, grid storage strategy (spec §39 — not decided here).
- Any code, schema, migration, API, or test change (none exist in this document).

### 1.3 The one-sentence contract

> GBA holds and releases contracted capacity according to decided lifecycle/wash rules; every consumption becomes **one canonical pickup record linked to a real reservation**, committed **atomically** with its counters; **Availability is the sole authority** for the sellable number and is invalidated by every GBA mutation through **consumed** domain events; all of it is **hotel-scoped**.

---

## 2. Canonical Vocabulary

**Vocabulary authority: S-3 = Option A (spec vocabulary canonical, confirmed 2026-09-30) `[DECIDED]` (TR-1.6).** Stored legacy values map through a published 1:1 translation applied at read boundaries now and in schema at Phase 11. No competing terminology may be used in rules, code comments, UI copy, or docs.

### 2.1 Core terms

| Canonical term | Definition (source) |
|---|---|
| **Group Booking** | The group-level reservation container that owns one or more blocks; `PROSPECT` by default, only `CONFIRMED`/`ACTIVE` bookings contribute to Availability (audit A1 `availability-source.adapter.ts:49`) |
| **Group Block** | A contracted room commitment with a group (MICE/wedding/corporate) for defined dates, room types, and negotiated rates (spec glossary `:27`) |
| **Block Code** | Alphanumeric block identifier, unique per hotel (TR-2.6) |
| **Allotment (Contract)** | A standing room quota agreement with a tour operator/bedbank/airline for ongoing seasonal availability (spec `:30`) |
| **Contract Code** | Alphanumeric contract identifier, unique per hotel (spec `:31`; `uq_allotment_contracts_hotel_code`) |
| **Pickup** | The act of consuming allocated inventory; **the pickup record is the single canonical ledger** of that act (S-1 = A, TR-4.9) |
| **Pickup record** | The canonical association entity linking a real reservation to its block or allotment (spec T-9 `:871-900`; TR-9.2) — tables `group_pickups` / `allotment_pickups` |
| **Delegate** *(alias, non-canonical)* | Spec's term for an individual block pickup; canonical term is **pickup record** (S-1) |
| **Voucher** | The allotment consumption authorization artifact (wholesaler booking record); **not** a ledger — consumption without a resulting pickup record is impossible (TR-4.9) |
| **Reservation** | The first-class stay record owned by the Reservations domain; pickup-created reservations are first-class (TR-9.1) |
| **Master Folio** | The consolidated A/R ledger for group-block charges (spec `:39`); receives `ATTRITION_FEE` postings (TR-6.9) |
| **Contracted Quantity** | Rooms committed per date/room type (`contracted_qty` on block allocations; `quota` on allotment quotas) |
| **Daily Quota** | The allotment's per-date/per-room-type entitlement (`allotment_daily_quotas.quota`) |
| **Picked** | Rooms consumed by pickups (`picked_qty` / `picked` counters) |
| **Released** | Rooms returned to house via wash/release (`released_qty` / `released`) |
| **Remaining** | `quota − picked − released` (allotment) or `current_held_qty − picked_qty` (block) — **one formula per pool, evaluated after the stop-sale check** (TR-3.2, TR-7.4, S-4) |
| **Physical Inventory** | Rooms the hotel has (room master) — owned by Availability |
| **Sellable Availability** | The computed sellable number for a date/room type — produced **only** by Availability A1 (TR-1.2, TR-10.1) |
| **Allotment entitlement** | The contractual right to consume from a daily quota — distinct from physical inventory (§5.4) |
| **Shoulder Days** | Additional nights before/after the core block date range, modeled as ordinary allocations in the extended range (spec `:41`; D-13 base rule) |
| **Cut-off Date** | The date at which unpicked block rooms are washed to general inventory (spec `:36`, T-1) |
| **Wash** | The scheduled return of unpicked allocated rooms to general inventory — one event per block at cut-off; rolling windows for allotments (T-1/T-2, TR-6.1/6.2) |
| **Rolling Release** | Allotment mechanism: unpicked rooms wash back N days before arrival (`release_days_before`) for `ROLLING_RELEASE` contracts (spec `:37`, T-2) |
| **Release** | The manual operator return of **unpicked** held rooms (TR-5.1) — coexists with scheduled wash (TR-5.5) |
| **Stop Sale** | A **selling-permission** restriction on allotment contracts preventing new intake for date/category; never changes quantity (spec `:38`, TR-7.1/7.2) |
| **Attrition** | Contractual shortfall when pickup < threshold % of contracted, assessed at wash (spec `:35`, T-7, TR-6.7) |
| **Deduct Inventory (DEDUCT)** | Block policy that physically removes held rooms from transient sellable capacity when the block is in an eligible status (spec `:42`, TR-1.3) |
| **Non-Deduct (NON-DEDUCT / soft hold)** | Block policy that commits logically without reducing transient sellable capacity (spec `:43`) |
| **HARD (commitment)** | Contract commitment semantics that **does** reduce sellable capacity in A1 (A1 filters it, `availability-source.adapter.ts:49-50`) |
| **SOFT (commitment)** | Contract commitment semantics that does **not** suppress transient sell (A1 ignores it) |
| **FREE-SALE** | Canonical contract type: open sale up to a ceiling; no release needed (spec `:598`) |
| **GUARANTEED_BLOCK** | Canonical contract type: take-or-pay — **never washes** (spec `:1146`, TR-6.3) |
| **ROLLING_RELEASE** | Canonical contract type: washes N days before arrival if not picked (T-2, TR-6.2) |
| **ATTRITION_FEE** | Master-folio charge category for the attrition penalty (TR-6.9) |

### 2.2 Contract-type canonical vocabulary + stored-value mapping `[DECIDED]` (S-3 = A)

**Canonical type enum:** `ROLLING_RELEASE | GUARANTEED_BLOCK | FREE_SALE` (spec `:346`, `:989`).
**Canonical commitment semantics:** HARD (deducting) vs SOFT (non-deducting) — orthogonal to type.

Stored values today: `SOFT_QUOTA`, `HARD_COMMITMENT`, `FREE_SALE`, `GUARANTEED` + boolean `is_rolling_release` (`allocation-status.value-object.ts:52-55`, `allotment.aggregate.ts:457`). The published 1:1 mapping must satisfy these already-fixed folds:

- `HARD_COMMITMENT` ↔ **HARD** commitment semantics (A1's eligibility filter is stated against this term — TR-10.2),
- `GUARANTEED` → **GUARANTEED_BLOCK**,
- `is_rolling_release = true` folds into the **ROLLING_RELEASE** type,
- `FREE_SALE` → **FREE_SALE**,
- `SOFT_QUOTA` → **SOFT** commitment semantics.

The complete table is a publication artifact required at read boundaries before implementation consumes it; the **direction is decided (spec-canonical)** — re-deriving it is not permitted. Schema renaming is Phase 11 (TR-15.2).

---

## 3. Domain Ownership Boundaries

### 3.1 Fact-ownership table

| Fact | Owner (authoritative) | Everyone else |
|---|---|---|
| Reservation lifecycle & status | **Reservations** | GBA consumes events; never writes reservation status except through decided cancel paths (audit C-13 targets removal of unscoped writes) |
| Physical availability / sellable computation | **Availability** (A1 — sole authority, TR-10.1) | GBA never computes or publishes an availability number (TR-1.2) |
| Allotment entitlement (quota, picked, released) | **GBA / Allotment** | Availability reads it as an input fact; never mutates it |
| Block held capacity (contracted/held/picked/released) | **GBA / Block** | Availability reads it as an input fact |
| Consumption linkage (reservation ↔ block/allotment) | **Pickup record** (GBA) — the association truth (TR-9.2) | `reservations` GBA columns are secondary/denormalized read-optimization only (TR-15.7) |
| Guest stay lifecycle | **Reservation / Front Office** (as previously defined) | GBA applies D-11 identity order only when creating pickup reservations |
| Pickup lifecycle (ACTIVE/CHECKED_OUT/CANCELLED) | **GBA / Pickup** — derived from reservation events (D-4) | FO performs **no** direct GBA writes (L-13 closed) |
| Voucher lifecycle (ISSUED/USED/CANCELLED/EXPIRED) | **GBA / Voucher** | Reservation lifecycle governs whether consumption stands (TR-12.1) |
| Stop-sale state (selling permission) | **GBA / Allotment** | Availability respects selling permission as a fact input |
| Attrition assessment (shortfall/liability) | **GBA** (at wash, TR-6.7) | — |
| Financial posting (`ATTRITION_FEE`, ROOM charges) | **Existing financial/folio owner** via the wash transaction / folio port (TR-6.9; existing pickup auto-post `create-group-pickup.handler.ts:64-73` preserved) | GBA does not own the ledger (spec open question 2 — implementation scope) |
| Availability assertion (balances) | **Availability assertion engine (Phase 2)** | Pickup-created reservations must pass through it (TR-4.1); wiring mechanism `[DEFERRED]` |
| Legacy GBA tables / legacy `availability` counters | **Legacy** — RETAIN read-only (TR-15.1) | GBA never writes them (TR-15.1); they are never authoritative for any Phase 4 fact |

### 3.2 Duplicate-truth prevention rules

1. Exactly **one** producer of sellable availability (TR-1.2, TR-10.1) — A3 and F-18 must derive from it or be retired (D-6a = A).
2. Exactly **one** consumption ledger (TR-4.9) — two record models/two guard formulas are prohibited (S-1, S-4).
3. Exactly **one** source of truth for whether consumption stands: **the reservation** (TR-12.1, TR-12.4).
4. Exactly **one** association truth: the **pickup record** (TR-9.2); bare columns are transition aids (TR-15.7).
5. Every rule in this document is **hotel-scoped**; no fact may be read or mutated cross-hotel (TR-14.1–14.5).

---

## 4. Group Block Domain Model

### 4.1 Identity & ownership

- **Aggregate root:** `GroupBlock`; children: daily allocations, pickup records, wash-history, master-folio transactions (spec §20) `[SPEC-CARRIED structure, consistent with D-1 new world]`.
- **Keys:** `id` (UUID); `hotel_id` (**mandatory predicate on every read/write**, TR-14.1); `block_code` unique per hotel (TR-2.6 — generation defect F-12 fixed during implementation: code derived from actual count, not a non-matching UUID query); `group_booking_id` FK → `group_bookings` (each block belongs to exactly one group booking and one hotel, TR-2.6).
- **Date range:** `arrival_date` → `departure_date`, extended by shoulder days (§9); stay dates are half-open `[arrival, departure)` (current aggregate behavior, `group-block.aggregate.ts:227-232` `[SPEC-CARRIED]`).
- **Rate linkage:** `rate_plan_code` + `negotiated_rate` on the block; pickup rate consistency vs negotiated rate is a pickup invariant (TR-4.6, spec §22 rate consistency `[SPEC-CARRIED]`).

### 4.2 Inventory policy: DEDUCT vs NON-DEDUCT `[DECIDED]` (D-6, TR-1.3, TR-2.2)

| Policy | Behavior |
|---|---|
| **DEDUCT_INVENTORY** | When the block is in an inventory-holding status (`DEFINITE`/`OPEN_FOR_PICKUP`), its contracted quantities **reduce sellable capacity** via A1's read-side input (`gbaRemaining = contracted − picked − released`) |
| **NON-DEDUCT** | Logical commitment: contributes **no** reduction of sellable capacity (A1 filters `DEDUCT_INVENTORY` only, `availability-source.adapter.ts:49`) |

Held capacity is an **inventory fact**, not availability policy: how it combines into "sellable" is defined solely by Availability (TR-1.3, §11).

### 4.3 Daily allocations

- Natural key `(hotel_id, group_block_id, stay_date, room_type)` `[DECIDED foundation, schema `:17130`]`.
- Per-row counters: `contracted_qty`, `current_held_qty`, `picked_qty`, `released_qty`, `version`.
- Block rollups (`contracted_nights`, `picked_nights`, `released_nights`) recomputed through the aggregate (`recalculateNights()`); **raw, unguarded, event-less mutation of block state is prohibited** (TR-2.5) — the current shoulder/category raw-SQL paths are non-conformant findings (F-11, C-9/C-10) that this contract makes mandatory to fix.
- Guard: per `hotel_id` + date + room type, **`picked ≤ contracted − released` after every committed operation**; reducing an allocation below its picked quantity is rejected; removing an allocation with active pickups is rejected (TR-2.4).

**Counter meanings:**

| Counter | Meaning |
|---|---|
| `contracted_qty` | rooms committed for that night/room type |
| `current_held_qty` | rooms still held (minus released) |
| `picked_qty` | rooms consumed by pickups |
| `released_qty` | rooms returned via wash/manual release |
| domain `remaining` | `current_held_qty − picked_qty` |

### 4.4 Status lifecycle

See §13.1. Rules: status changes only through explicit validated transitions (TR-2.1); only `DEFINITE`/`OPEN_FOR_PICKUP` hold inventory (TR-2.2); pickup requires an eligible state (TR-2.3); blocks start `DRAFT` (schema default) and must gain a first-class `DRAFT→TENTATIVE` step (D-15).

### 4.5 Cut-off, release policy, wash

- `cutoff_date` (DATE) drives the scheduled cut-off wash (§8, TR-6.1/6.4) `[DECIDED]`.
- Manual release coexists as operator override (TR-5.5) with guard `release ≤ held − picked` (TR-5.1).
- Until the scheduled path ships, cut-off/release fields are **non-authoritative display metadata**; once shipped they are authoritative — never stored-but-ignored (TR-6.5).
- Scheduler technology `[DEFERRED]` (TR-6.6).

### 4.6 Block types / GUARANTEED / ROLLING / FREE-SALE interaction `[DECIDED]` (D-3, S-3)

- The canonical type triad (§2.2) governs **wash eligibility by type**: `GUARANTEED_BLOCK` never washes — excluded from cut-off and rolling paths wherever the type is present (TR-6.3, D-3 selected rule); `ROLLING_RELEASE` washes N days pre-arrival; `FREE_SALE` needs no release (open sale up to ceiling, spec `:598` `[SPEC-CARRIED]`).
- Group-block cut-off wash (T-1) applies to blocks per their `cutoff_date` (TR-6.1).
- HARD/SOFT commitment semantics govern **availability contribution** (§11), not wash.

### 4.7 Attrition & shoulder

Covered by §10 (D-14 = A) and §9 (D-13 = A) — both fully decided; no separate block rules exist beyond those sections.

---

## 5. Allotment Domain Model

### 5.1 Identity & aggregate

**Aggregate root:** `AllotmentContract`; children: daily quotas, vouchers, stop sales, pickup records, release logs (spec §20 `[SPEC-CARRIED]`, matching new-world tables). Keys: `id` (String UUID), `hotel_id` (mandatory predicate), `allotment_code` unique per hotel.

### 5.2 Daily quota

- Natural key `(hotel_id, allotment_id, stay_date, room_type)` (schema `:17215`) with counters `quota`, `picked`, `released`, `stop_sale_active`, `net_rate`, `version`.
- **Single guard formula for all intake paths:** per `hotel_id` + date + room type, **`picked + released ≤ quota`** after every committed operation; consumption uses `remaining = quota − picked − released` (TR-3.2, S-4).
- Contract rollups `total_daily_quota` / `total_picked` / `total_released` are part of the same committed unit as their quota rows (invariant-satisfaction after commit, TR-11.3 — the audit's C-5 double-write is non-conformant).

### 5.3 Contract lifecycle & intake eligibility

`DRAFT → ACTIVE → SUSPENDED ⇄ ACTIVE`, plus `→ CLOSED` and `→ EXPIRED` (TR-3.3; spec T-6 `ON_HOLD` naming maps to `SUSPENDED` via TR-1.6's published mapping; spec `TERMINATED` maps to the `CLOSED` terminal). Consumption accepted **only while `ACTIVE` and not expired** (TR-3.3, TR-13.1). Expired/closed contracts contribute **zero** commitment via A1's validity-window/status filter (TR-13.2).

### 5.4 Physical availability vs allotment entitlement (explicit distinction)

| | **Physical availability** | **Allotment entitlement** |
|---|---|---|
| What | Rooms the hotel physically has, combined into sellable by Availability (A1) | The contractual right to consume a daily quota |
| Owner | Availability | GBA |
| Unit | room type × date | quota rows: `quota − picked − released` |
| Overbooking | Only via transient `overbooking_limits` allowance (TR-10.5) | **Never**: `quota ≤ physical` (TR-1.5) |
| Interaction | A1 **subtracts** HARD-commitment `allotmentRemaining` as a read-time input fact (A1 `:50,:82`); a pickup reduces both the entitlement (picked++) and — via the assertion path — transient capacity (TR-4.1) | Entitlement guards are quota-internal and unchanged by S-2 (TR-3.5) |

### 5.5 S-2 rule: contracted daily quota MUST NOT exceed physical inventory `[DECIDED]` (S-2 = A, TR-1.5, TR-3.5)

1. `quota > physical` for any date/room type is **prohibited** at contract **create** and **set-quota** (guard to be added in implementation — currently absent).
2. **Pre-existing violations** (verified only as flagged facts): surfaced as flagged facts under TR-10.4/10.5 posture — **never silently rewritten**.
3. **Overbooking relationship:** the sole overbooking facility remains the transient `overbooking_limits` (hotel + room type + date → allowance feeding A1). Allotment contract overcommit is a different concept, forbidden; it never shares, feeds, or mimics that allowance (TR-10.5).
4. Pickup guards remain quota-internal (TR-3.5); A1 math unchanged — the `Math.max(0, …)` clamp ceases to be a de-facto enforcer because input state can no longer arise from contract writes (S-2 selected rule §2).
5. Adopting overbooking later **requires re-deciding TR-1.5** (explicitly not allowed to drift).

### 5.6 Stop sale

§7 rules apply here: selling-permission only, allotment-contract-only scope, quantity untouched (TR-7.1–7.5, S-4).

---

## 6. Pickup / Voucher Domain

### 6.1 The canonical consumption model `[DECIDED]` (S-1 = A, D-10 = B, D-9)

```
Voucher path:   issue (quota held: picked++)  →  consume (reservation created/linked + pickup record, status USED)
Direct path:                                             create pickup (reservation created + pickup record, picked++)
Both paths → ONE canonical pickup record, linked to a REAL reservation, guarded uniformly, committed atomically.
```

- **S-1 = A:** one canonical pickup record type for both entry paths; vouchers are authorization artifacts; **consumption without a resulting pickup record is impossible** (TR-4.9). Direct/manual intake remains a supported capability (verified live, `AllotmentDetailView.tsx:340`).
- **D-10 = B:** voucher consumption **creates (or links an already-created) real reservation as part of consume**; fabricated identifiers (`RES-…`) are prohibited; `reservationId` is a real id in this hotel's data or null/absent (TR-3.4, TR-4.4, TR-9.3).
- **D-9:** reservation row + pickup record + counter changes (+ folio effects where part of the operation) commit **together or not at all** (TR-4.2, TR-9.5).
- **Guard order on every intake path:** (1) contract/block eligibility state, (2) **stop-sale check** (allotment paths), (3) remaining-per-TR-3.2 — stop sale **before** quantity (TR-7.4, TR-4.5), then (4) containment invariants TR-4.6, then (5) two-layer validation §7.1 (TR-4.1).

### 6.2 Counter timing (exactly-once, TR-11.4)

| Path | `picked` increments | `picked` decrements |
|---|---|---|
| Voucher issue | **at issue** (S-1 confirmed: quota held at issue, stop sale checked) | ISSUED-voucher cancel (TR-12.5); reservation-cancel restore (TR-12.1/12.4) |
| Voucher consume | **no counter change** (already held at issue) — consume adds reservation + pickup record | — |
| Direct pickup | **at pickup**, inside the atomic unit | pickup cancel / reservation cancel (TR-12.1/12.3) |
| Checkout | **never** — the room was consumed (TR-4.8) | — |

### 6.3 Voucher lifecycle (entity states) & legal transitions

Stored states: `ISSUED → USED → CANCELLED`, plus `ISSUED → EXPIRED` (TR-13.3).

| From | To | Condition / rule |
|---|---|---|
| — | `ISSUED` | Issue guarded: contract `ACTIVE`, not expired, stop-sale pass, remaining pass (TR-3.3, TR-7.4, TR-3.2); `picked++`; code uniqueness `(hotel_id, allotment_id, voucher_code)` |
| `ISSUED` | `USED` | Consume only: creates/links **real** reservation (D-10-B) + creates pickup record (S-1); atomic (D-9); **idempotent** — same voucher twice returns the existing reservation, never a duplicate (spec §32 `[SPEC-CARRIED]`, consistent with TR-11.4) |
| `USED` | `USED` | **Illegal** (duplicate consumption) |
| `ISSUED` | `CANCELLED` | Restores held quota (`picked--`, floor-bounded) (TR-12.5) |
| `USED` | `CANCELLED` | **Quota restores iff the linked reservation is cancelled** — see §6.4 (TR-12.4); voucher state alone never decides quota |
| `ISSUED` | `EXPIRED` | Past validity window: cannot be consumed; any quota outcome limited to issued-state handling (TR-13.3) |
| `CANCELLED` / `EXPIRED` / any re-move from them | — | **Illegal** — deterministic rejection (INV-13) |

**Reservation-side stages (not voucher states):** the spec's conceptual post-intake stages (`CONFIRMED → CHECKED_IN → CHECKED_OUT`, `NO_SHOW`, spec `:710-714`, `:1108-1116`) belong to the **linked reservation** and are reflected onto the **pickup record** by the D-4 event-consumer derivation (§7.2). Mapping: spec `INTAKE/CONFIRMED` ≈ `USED` + reservation created; `CHECKED_IN/CHECKED_OUT/NO_SHOW` = reservation-owned states. *(Supersession #2, §20.2.)*

### 6.4 S-5 rule (USED-voucher cancellation) `[DECIDED]` (S-5 = A, TR-12.4)

> Cancelling a **USED** voucher: quota restores **iff** the linked reservation is cancelled. The reservation lifecycle is the single source of truth (TR-12.1).

1. Reservation live ⇒ quota stays consumed (restoring would over-sell and silently break S-2=A's effective cap).
2. Reservation cancelled ⇒ restore (same bounds as issued-branch restore), riding the D-9 atomic path with D-10 linkage.
3. **Post-intake shape:** voucher cancellation after consumption is handled as a **regular reservation cancellation** (spec `:725`; before check-in restore at `:723`) — the command delegates to reservation-cancel semantics rather than writing quota itself.
4. Rejected: "always restore" (over-sell under live reservation) and "never restore" (permanent quota burn).
5. Anomalous `USED` rows with `reservationId == null`: defensive handling rides D-10; existence is a live-DB question — **SOURCE-SILENT, no guessed rule** (implementation must surface, not invent).
6. No schema/migration impact. Entity-guard changes to permit the decided flow are implementation scope (D-10/D-4 note).

### 6.5 Duplicate-consumption prevention

Voucher code uniqueness (DB unique key) + intake idempotency (spec §32) + exactly-once counters (TR-11.4) + real-linkage requirement (D-10). A second consume of a `USED` voucher is rejected deterministically (§13.2).

---

## 7. Reservation ↔ GBA Integration

### 7.1 How a pickup creates/links a reservation `[DECIDED]` (D-7, D-10, D-9, D-11)

1. **Two-layer validation, never either alone** (TR-4.1): (a) pool layer — capacity on every stay date + room type; (b) reservation layer — the created reservation passes the **same** availability/assertion lifecycle as any other reservation (assert-at-write vs read-time mechanism `[DEFERRED]`, D-7 note).
2. **Reservation identity:** pickup-created reservations are first-class (TR-9.1); `source`/`pickup_type` may label origin but never change rules; status honors the reservation domain's confirmation semantics — **not** hardcoded (TR-9.7, mechanism `[DEFERRED]`).
3. **Association:** pickup record carries the real `reservation_id` (TR-9.2/9.3); bare columns on `reservations` remain secondary read-optimization (TR-15.7).
4. **Guest resolution (D-11 = C amended, TR-4.10, TR-9.6):** order = (1) explicit Guest ID (hotel-scoped exact; authoritative only if it resolves in-hotel) → (2) Passport ID (hotel-scoped exact) → (3) Email (hotel-scoped exact) → (4) **create new guest**. Every lookup includes `hotel_id`; `full_name ILIKE`/fuzzy/name matching prohibited; no `LIMIT 1` name binding; Passport-vs-Email resolving to different guests = **conflict requiring explicit handling**, never a silent choice. No rule beyond this order is decided; caller-contract changes are implementation scope.
5. **Idempotency expectations:** counter mutations exactly-once per business event (TR-11.4); voucher intake idempotent (spec §32); request-level dedup mechanics `[DEFERRED]`.

### 7.2 Reservation lifecycle drives pickup state (D-4 = B structure + C semantics) `[DECIDED]`

| Reservation event | Pickup effect | Counter effect | Availability effect | Financial effect |
|---|---|---|---|---|
| `reservation.cancelled` | pickup → `CANCELLED`, association unlinked per TR-9.2 | **restore exactly once** (TR-12.1, TR-12.3) | restored capacity invalidates A1 atomically (TR-12.6, TR-10.3) | none decided beyond existing flows |
| `reservation.checked_out` | pickup → `CHECKED_OUT` (consumed) | **unchanged** (TR-4.8) | none (room was sold) | none |
| (context) FO check-out handler direct GBA writes **removed** (L-13 closed); failures retry via queue instead of being swallowed (D-4 confirmed rule) | | | | |

Delivery mechanism: D-5's mandatory **GBA-side reservation-event consumer**. Producer = Reservations/FO context; consumer = GBA. Idempotency: consumer idempotent (TR-11.4); at-least-once delivery assumed by the outbox→queue→handler mechanism (unchanged per D-5); **no cross-event ordering guarantee is decided** — consumers must be order-tolerant (no ordering requirement is established by any resolved decision).

### 7.3 Other reservation interactions

| Interaction | Rule | Status |
|---|---|---|
| Check-in | Voucher redemption = reservation check-in (spec `:718`); pickup counters unchanged; room assignment belongs to FO | `[SPEC-CARRIED]`; no counter rule contradicted |
| No-show | Quota **not** restored (consumption stands; release window passed, spec `:728`, `:1116`) | `[SPEC-CARRIED]` — pickup record status after no-show: **SOURCE-SILENT** (no invented rule) |
| Checkout | Pickup → `CHECKED_OUT`, counters unchanged | `[DECIDED]` TR-4.8/9.4 |
| Extend / overstay | No resolved rule for stay extensions changing pickup containment post-creation. Containment (TR-4.6) applies **at pickup creation**. | **SOURCE-SILENT** — flagged for Implementation Plan; no rule asserted |
| Modify / change room type/rate | Reservation domain owns modify; no resolved rule re-derives pickup counters on modify. | **SOURCE-SILENT** — no rule asserted |
| Pickup cancellation (command) | Restores counters exactly once, unlinks association (TR-12.3) | `[DECIDED]` |
| Reservation already cancelled when voucher cancelled | Restore already applied by reservation-cancel path; duplicate application prohibited (TR-11.4) | `[DECIDED]` |

---

## 8. Block Lifecycle / Cut-off / Wash / Release

### 8.1 Cut-off

At the block's **cut-off date**, unpicked allocated rooms return to general inventory in a **single wash event per block**, logged with date, count, reason (TR-6.1, T-1). Timing is **date-granularity** (DATE columns): the wash applies **from the start of the applicable day**, not an arbitrary clock time (TR-6.4).

### 8.2 Wash — complete semantics `[DECIDED]` (D-3 = A)

| Aspect | Rule |
|---|---|
| **What is washed** | **Unpicked** rooms (`held − picked`, floor ≥ 0) of the block's allocations across the full extended range (core **+ shoulder dates**, no exclusion filter — TR-8.6b) |
| **When — blocks** | At `cutoff_date` (cut-off wash, T-1, TR-6.1) |
| **When — allotments** | `ROLLING_RELEASE`: N days before arrival per `release_days_before` (T-2, TR-6.2); window mechanics `[SPEC-CARRIED]` (spec `:600-609`: today → today + releaseDays, per category) |
| **Excluded** | `GUARANTEED_BLOCK` **never washes** — excluded by type from both paths (TR-6.3); expiry does **not** trigger wash (TR-13.4) |
| **Which quota released** | The unpicked remainder: `released += unpicked` (block `released_qty`, allotment `released`), so post-wash `picked + released = quota` for the washed range and A1's `gbaRemaining`/`allotmentRemaining` input reads 0 — while `picked` stays (rooms consumed stay consumed) |
| **Which dates affected** | The wash range: block = whole extended range at cut-off (one event); allotment = dates in the rolling window per T-2 |
| **Counter effects** | TR-2.4/TR-3.2 inequalities must hold after commit; `picked` never changes in a wash |
| **Transaction boundary** | **One atomic unit**: counter updates + wash log + attrition assessment + penalty posting in a single commit (D-9, TR-11.6, TR-6.7 — spec `:1524-1530`) |
| **Idempotency** | Repeated execution for the same range is a **no-op** (spec §32 `[SPEC-CARRIED]`, consistent with TR-11.4) |
| **Event propagation** | Emits `CutOffWashExecuted` / `AllotmentWashExecuted` (TR-6.5) through D-5 consumers → Availability invalidation (TR-10.3). **Event-less wash is prohibited** |
| **Audit log** | Timestamp, actor (system or user), rooms released (total + per category), target dates, reason, before/after state (spec `:611-621` `[SPEC-CARRIED]`) |
| **Scheduler** | Technology/mechanism `[DEFERRED]` (TR-6.6). Trigger semantics are decided; the transport (e.g., platform job pattern) is not |
| **Data honesty** | Until the scheduled path ships, cutoff/release fields are non-authoritative display metadata; once shipped they are authoritative (TR-6.5) |

### 8.3 Manual release `[DECIDED]` (TR-5.1–5.5, D-12)

- Returns **unpicked** held rooms only: `release quantity ≤ held − picked` (the current `canRelease ≤ held` allowance is prohibited); releasing picked rooms requires cancelling the consuming reservation (TR-5.1).
- Released rooms return to sellable through Availability via immediate input invalidation (TR-5.2, D-5 events); atomic + evented (TR-5.2, TR-11.6).
- Impossible against `DRAFT`/`CLOSED`/`CANCELLED` blocks and non-`ACTIVE` allotments; hotel-scoped (TR-5.3).
- Canonical API verb is **`POST`** (D-12; frontend caller corrected, no `PUT` alias — implementation fix of an already-decided contract).
- Coexists with scheduled wash as operator override (TR-5.5).

---

## 9. Shoulder Days (D-13 = Option A — uniform)

`[DECIDED]` — base rule + three differentiators, all confirmed 2026-09-30:

1. **Ordinary allocations in the extended range** (TR-8.1): same entities, guards, events, atomicity, hotel scope as core days.
2. **Declared quantities** — silent qty-0 placeholders prohibited; shoulder counters accumulate (no opposite-direction overwrite) (TR-8.2).
3. **Removal obeys the no-removal-with-active-pickups guard** and recomputes block rollups; unguarded deletion prohibited (TR-8.3, closes F-11).
4. **Mutations emit domain events** and invalidate Availability like any allocation change (TR-8.4).
5. **Shoulder bookkeeping columns** (`shoulder_days_before/after`) are declared schema — reproducible in a fresh environment (TR-8.5, rides D-2).
6. **Uniform differentiators (confirmed A):**
   - **P1 — attrition base: included** (TR-8.6a, no core/shoulder filter in T-7 math),
   - **P2 — cut-off/wash: identical** — scheduled wash covers the extended range with no exclusion (TR-8.6b),
   - **P3 — pickups: unrestricted** — identical guards to core dates (TR-8.6c, TR-4.7).
   Differentiated/buffer treatment rejected; revisiting would require re-deciding the base rule too.

---

## 10. Attrition (D-14 = Option A)

`[DECIDED]` — implemented per spec T-7, assessed **exactly once**:

| Aspect | Rule |
|---|---|
| **When** | At the block's scheduled cut-off wash only (TR-6.7). **No** attrition evaluation at pickup, booking, availability, or expiry time |
| **Transaction** | Inside the **same wash transaction** — counters + wash log + penalty in one commit (TR-6.7, spec `:591-592`, `:1524-1530`) |
| **Policy** | Single percentage per block, **default 80%**, uniform across room types (TR-6.8, spec `:560-566`). Rejected: VO default 85, `per_day_minimum`/`cumulative`/`room_type_specific` types incl. hardcoded-0.85 — richer types require a new decision |
| **Base** | All contracted room-nights in the wash range **including shoulder days** (TR-6.8, TR-8.6a) |
| **Math** | `minimumRequired = ceil(contracted × threshold / 100)`; `shortfall = max(0, minimumRequired − picked)`; `meetsThreshold = picked ≥ minimumRequired` (TR-6.8, spec `:554-560`) |
| **Penalty** | If `shortfall > 0`: `liabilityDue = shortfall × negotiated rate`, posted to the block's **master folio** as **`ATTRITION_FEE`** **within the wash transaction** — never a separate async write (TR-6.9) |
| **Scope** | Group **blocks** only; allotment rolling release has no attrition (TR-6.7) |
| **Read-only discipline** | Assessment is **read-only w.r.t. inventory** — never mutates availability/pickup counters, never guards a pickup (TR-6.7). Pre-wash shortfall reports are read-side only and carry **no authority** |
| **Single-shot** | Follows from the one-wash-event design (TR-6.1) — no double-penalty path (TR-6.7) |
| **Calculation vs posting ownership** | The **calculation** (shortfall/liability) is owned by GBA (this section). The **financial posting** lands on the master folio owned by the existing financial/folio owner; **ledger ownership of that posting is implementation scope** (spec open question 2) `[DEFERRED]` |
| **Threshold bounds** | `50% ≤ threshold ≤ 100%` `[SPEC-CARRIED]` (spec §22 — not contradicted, not re-decided) |

---

## 11. Availability Integration

### 11.1 Authority `[DECIDED]` (D-6, D-6a = A)

- **A1 (the Availability snapshot/assertion computation) is the sole producer of sellable availability** (TR-10.1, TR-1.2). No second inventory engine exists; GBA never publishes its own availability figure (TR-1.2).
- **A3 (Activities matrix) is reimplemented on top of Availability** (D-6a = A). A labeled interim view (option B) is permitted **only** as an explicitly-labeled transitional state whose label states "derived view, not sellable availability".
- **F-18** (`GET /group-bookings/available-rooms`, date-blind) is retired under the D-6 rule.
- Eligibility filters live in **one place**, defined once and consumed everywhere (TR-10.2): DEDUCT policy, booking-status set (`CONFIRMED`/`ACTIVE`), soft-delete exclusion, block-status whitelist (`DEFINITE`/`OPEN_FOR_PICKUP`), contract-type/commitment + validity-window checks, per-hotel (TR-14.4).
- **Four tracked fact kinds** combine into sellable — physical quantity, selling permission, reservation commitment, history (TR-1.1) — only their combination is single-sourced.

### 11.2 Data flow (explicit)

```
GBA mutation (block/quota/pickup/release/wash/stop-sale/shoulder/status)
   |  guarded aggregate -> domain event (TR-2.5 / TR-6.5) -- commits atomically
   v
Mandatory consumers (D-5):
   (a) Availability invalidation -- affected hotel/room-type/date facts recomputed (TR-10.3)
   (b) GBA reservation-event consumer <- reservation.cancelled / reservation.checked_out (D-4)
   v
Availability A1 recomputes sellable:
   consumption = reservationConsumption (A6)
               + gbaRemaining        (block: contracted - picked - released, eligible blocks only)
               + allotmentRemaining  (HARD-commitment, ACTIVE, in-validity contracts only)
               - overbookingAllowance (transient overbooking_limits only)
   integrity checks -> UNRESOLVED rows flagged with provenance (TR-10.4), never trusted silently
   v
Sellable availability published (snapshot / assertions / UI views incl. rebuilt A3)
```

- Read-time subtraction: a block/allotment ceases to contribute the moment it leaves eligibility (status change, soft-delete, expiry) — invalidation events exist to kill staleness; **stale-by-design is prohibited** (TR-10.3).
- **Overbooking:** if ever adopted, the overcommit appears as an **explicit flagged fact** inside the authority — never a second number, never a silent clamp (TR-10.5).
- **Provenance & unresolved-flagging** are part of the contract (TR-10.4).
- **Pickup consults this authority** (T-3 duty; TR-10.6) — the two-layer reservation check (§7.1); the wired port shape `[DEFERRED]` (D-7 note, dead `IInventoryCommitmentPort` replaced).
- **Legacy A4 counters** (`availability` table via CRS engine): a disjoint legacy writer — GBA never reads or writes it (audit §3); not part of this flow (§18).
- **Assertion engine (A5):** pickup-created reservations must flow through it (TR-4.1); wiring `[DEFERRED]` (L-14/A5 gap).

---

## 12. Concurrency and Transaction Rules

**Level of statement:** business invariants only. **No locking syntax, no mechanism.** The concrete mechanism (conditional-version retry vs load-time lock) is `[DEFERRED]` — **IMPLEMENTATION-PLAN DEFERRED** (TR-11.7, D-8). A deferred mechanism may never be quoted as a business rule.

### 12.1 Standing invariants `[DECIDED]` (D-8)

1. **No lost updates** (TR-11.1): concurrent counter writes never merge; exactly one wins, the other fails or retries.
2. **Conflict is an error** (TR-11.2): surfaced as `CONFLICT` to the caller; silent last-write-wins prohibited.
3. **Core inequality after every commit** (TR-11.3): block `picked ≤ contracted − released`; allotment `picked + released ≤ quota` (per hotel/date/room type).
4. **Exactly-once counter mutation per business event** (TR-11.4): retries idempotent; double-application prohibited.
5. **Version monotonicity** (TR-11.5): staleness detection never rolls backwards; writes from stale state never commit.

### 12.2 Transaction boundaries per operation

| Operation | Atomic unit contents | Source |
|---|---|---|
| **Pickup — block** | reservation row + pickup record + block/allocation counters (+ folio charge if part of the operation) | D-9, TR-4.2, TR-9.5 |
| **Pickup — allotment (direct)** | reservation row + pickup record + quota counters | D-9, TR-4.2 |
| **Voucher issue** | voucher row + quota `picked++` + stop-sale/remaining guard pass | S-1 (hold at issue), TR-11.3/11.4 |
| **Voucher consume** | reservation created/linked + pickup record + voucher `USED` (no counter change) | D-10-B, S-1, D-9 |
| **Reservation cancellation cascade** | pickup → CANCELLED + counter restore (exactly once) + unlink | TR-12.1, TR-12.3, D-9 discipline |
| **Voucher cancellation** | voucher state + (conditional) restore per TR-12.4, or delegation to reservation-cancel | S-5 clause 2 |
| **ISSUED-voucher cancel** | voucher state + `picked--` restore | TR-12.5 |
| **Wash (cut-off/rolling)** | counters + wash log + attrition assessment + `ATTRITION_FEE` posting | D-3, D-14, TR-6.7/6.9, spec `:1524-1530` |
| **Manual release** | counter change + domain event | TR-5.2, TR-11.6 |
| **Attrition posting** | *inside the wash unit* — never a separate async write | TR-6.9 |
| **Block status transitions** | status change + domain event + rollup effects | TR-2.5, TR-11.6 |
| **Quota changes (set-quota/batch)** | quota rows + contract rollup such that TR-11.3 holds after commit; multi-step partial commit of one business event prohibited (audit C-5 is the target defect) | TR-11.3/11.4 applied to one business event |
| **Shoulder add/remove** | allocations + block date/counter bookkeeping + rollup recompute + event | TR-8.1–8.4, D-9 |

### 12.3 Cross-cutting behavior

- **Atomicity:** partial commit is prohibited in every failure mode; compensating scripts are not a substitute for atomicity (D-9 — today's swallowed `rollbackQuota` is explicitly not a guarantee).
- **Idempotency:** every retry/duplicate converges to one application (TR-11.4); voucher intake and repeated wash additionally per spec §32 `[SPEC-CARRIED]`.
- **Ordering:** none decided across events (§7.2). Lock ordering / contention strategy `[DEFERRED]` (TR-11.7).
- **Hotel isolation:** every statement — including raw SQL — carries `hotel_id` in its predicate; mutation by bare id prohibited (TR-14.1; closes the 4 unscoped statements, audit §4).
- **Duplicate operations:** deterministic rejection or no-op per §15; never double-applied.
- **Partial failure:** whole unit rolls back; failure surfaces to the caller (no silent swallow — applies to D-4's consumer retries and all guards).

---

## 13. State Machines

Conventions: **Owner** = context performing/authorizing the transition. Every GBA transition emits its domain event and commits atomically (TR-2.5, TR-11.6).

### 13.1 Group Block (owner: GBA; actors `[SPEC-CARRIED]` where noted)

States: `DRAFT`, `TENTATIVE`, `DEFINITE`, `OPEN_FOR_PICKUP`, `CLOSED`, `CANCELLED`.
Legal chain: `DRAFT → TENTATIVE → DEFINITE → OPEN_FOR_PICKUP → CLOSED`; `→ CANCELLED` permitted from **any pre-closed state**; `DRAFT→TENTATIVE` is **first-class** (TR-2.1). Implicit/automatic change prohibited; widening A1 eligibility to compensate prohibited.

| Transition | Trigger / validation | Counter effect | Reservation effect | Availability effect | Financial effect |
|---|---|---|---|---|---|
| `→ DRAFT` | create command (schema default; grid initialized) | grid created; **holds nothing** | none | **no contribution** (TR-2.2) | none |
| `DRAFT → TENTATIVE` | explicit lifecycle op (group name/dates/contact provided `[SPEC-CARRIED]`) | none (still holds nothing) | none | no contribution | none |
| `TENTATIVE → DEFINITE` | explicit op; contract signed/deposit `[SPEC-CARRIED]` | none (allocations unchanged) | none | **begins contributing** `gbaRemaining` (DEDUCT, TR-1.3/2.2) | none |
| `DEFINITE → OPEN_FOR_PICKUP` | explicit op (pickup readiness) | none | pickups now eligible (TR-2.3) | still contributes (whitelist includes both states) | none |
| `OPEN_FOR_PICKUP → CLOSED` | explicit op (consumed/finished `[SPEC-CARRIED]`); post-wash typical | history preserved; `CLOSED` holds nothing (TR-2.2) | existing pickup/reservation records unaffected | **stops contributing** immediately | none decided |
| `*pre-closed → CANCELLED` | explicit op (contract cancelled) | held rooms released to inventory **through Availability** (TR-12.2) | existing records preserved; new pickup rejected (TR-2.3) | stops contributing immediately | none decided (master folio closure `[SPEC-CARRIED]` only) |

Command/route/permission surface for these transitions: `[DEFERRED]` (D-15 note — implementation plan).

### 13.2 Voucher (owner: GBA/Allotment)

States: `ISSUED`, `USED`, `CANCELLED`, `EXPIRED`. Full legal/illegal matrix in §6.3.

| Transition | Trigger | Counter effect | Reservation effect | Availability effect | Financial effect |
|---|---|---|---|---|---|
| `→ ISSUED` | issue command; guards: ACTIVE, not expired, **stop-sale first**, remaining (TR-4.5/7.4) | `picked++` (held at issue) | none yet | quota input changes → invalidate (TR-10.3) | none |
| `ISSUED → USED` | consume; **creates/links real reservation** (D-10-B) + pickup record (S-1) | **unchanged** | reservation created (first-class, §7.1) | via reservation/assertion path (TR-4.1) | per reservation flows |
| `ISSUED → CANCELLED` | cancel | `picked--` restore (TR-12.5) | none | invalidate (TR-12.6) | none |
| `USED → CANCELLED` | cancel; **iff linked reservation cancelled ⇒ restore** (TR-12.4); else delegate to reservation-cancel (spec `:725`) | conditional restore (§6.4) | reservation is the truth (TR-12.1) | invalidate only when restored | none decided |
| `ISSUED → EXPIRED` | validity window passes (TR-13.3) | no consumption possible; issued-state handling at most | none | contract stops contributing via validity filter (TR-13.2) | none |
| any other move | — | **illegal → deterministic rejection** | — | — | — |

### 13.3 Pickup record (owner: GBA/Pickup)

States: `ACTIVE`, `CHECKED_OUT`, `CANCELLED` (existing status vocabulary, `group_pickups.status` default `ACTIVE`).

| Transition | Trigger (owner) | Counter effect | Reservation effect | Availability effect | Financial effect |
|---|---|---|---|---|---|
| `→ ACTIVE` | pickup create (GBA) or voucher consume (GBA) — inside atomic unit (§12.2) | `picked++` at pickup / already held at issue | reservation created/linked (real id) | two-layer check passes; inputs invalidate | ROOM auto-post where existing behavior applies (`create-group-pickup.handler.ts:64-73` preserved) |
| `ACTIVE → CHECKED_OUT` | `reservation.checked_out` event (Reservations/FO owns trigger; **GBA owns the write** — D-4) | **unchanged** (TR-4.8) | checkout completed | none | none |
| `ACTIVE → CANCELLED` | (a) `reservation.cancelled` event (D-4) or (b) cancel-pickup command (TR-12.3) | restore exactly once (TR-12.1/12.3) | unlinked per TR-9.2 | restored capacity invalidates (TR-12.6) | none decided |
| `CHECKED_OUT → ?` | cancellation after checkout (rare path) | — | — | — | **SOURCE-SILENT — no rule asserted** |
| duplicate cancel / duplicate consume | any | idempotent no-op / rejection (TR-11.4, §6.5) | — | — | — |

### 13.4 Allotment contract (owner: GBA/Allotment)

States (TR-3.3): `DRAFT`, `ACTIVE`, `SUSPENDED`, `CLOSED`, `EXPIRED`; `SUSPENDED ⇄ ACTIVE`. (Spec variant names `ON_HOLD`=`SUSPENDED`; `TERMINATED`→`CLOSED` terminal — mapping via TR-1.6 published mapping.)

| Transition | Trigger | Counter effect | Reservation effect | Availability effect |
|---|---|---|---|---|
| `→ DRAFT` / `→ ACTIVE` | create + activate (existing behavior: `create-allotment.handler.ts:49`) | quotas exist; may contribute | intake permitted | HARD + ACTIVE + in-validity contributes |
| `ACTIVE → SUSPENDED` | partner hold `[SPEC-CARRIED actor]` | none | intake **rejected** (TR-3.3: consume only while ACTIVE) | stops contributing (A1 status filter) |
| `SUSPENDED → ACTIVE` | hold lifted | none | intake resumes | contributes again |
| `* → EXPIRED` | `validity_end` passed (TR-13.1) | none (expiry does not wash, TR-13.4) | intake blocked | contributes **zero** (TR-13.2) |
| `* → CLOSED` | early termination / close (TR-12.2) | history preserved; no hard deletion | intake blocked | contributes zero |

### 13.5 Stop sale (owner: GBA/Allotment)

States: `APPLIED → LIFTED` (spec `:1041-1045`, `:1101-1106` `[SPEC-CARRIED]`, consistent with S-4).

| Transition | Trigger | Counter effect | Availability effect |
|---|---|---|---|
| `→ APPLIED` | revenue manager; date range + category (or ALL) + reason; **per hotel + date + category** (TR-7.5) | **none — quantity untouched** (TR-7.2) | display remaining reads 0 **for selling display only** (TR-7.2) |
| `APPLIED → LIFTED` | lift; restores **selling only** (TR-7.3) | none | display returns to actual remaining |

Scope: **allotment contracts only** — group-block pickups are never stop-sale restricted (TR-7.1). Existing reservations unaffected (spec `:652` `[SPEC-CARRIED]`).

### 13.6 Reservation interaction ownership summary

| Transition | Owner of transition | GBA receives |
|---|---|---|
| create/confirm/guarantee | Reservations (pickup path may create — §7.1) | pickup record created in same atomic unit |
| check-in / no-show | Reservations / FO | none decided (§7.3 — SOURCE-SILENT spots marked) |
| check-out | FO | `reservation.checked_out` → pickup `CHECKED_OUT` |
| cancel | Reservations | `reservation.cancelled` → pickup `CANCELLED` + restore |
| modify/extend | Reservations | **no resolved rule** — SOURCE-SILENT (§7.3) |

---

## 14. Business Invariants

**Business invariants (binding).** Implementation mechanisms are *not* invariants — see the mechanism column.

| # | Invariant | Source | Mechanism |
|---|---|---|---|
| INV-1 | **Hotel isolation:** every read/write (incl. raw SQL) carries `hotel_id`; no bare-id mutation; privileged-only header override; dev bypass never on real data | TR-14.1–14.5 (S-6) | `[DEFERRED]` (RLS etc. out of scope) |
| INV-2 | **Quota ≤ physical:** `quota > physical` prohibited at create/set-quota; historical violations flagged, never rewritten | TR-1.5, TR-3.5 (S-2=A) | guard shape `[DEFERRED]` |
| INV-3 | **No duplicate consumption:** one consumption event → one increment; voucher intake idempotent; `USED` never re-consumed | TR-11.4, spec §32 | `[DEFERRED]` |
| INV-4 | **Reservation is the source of truth for used-voucher consumption** (and consumption generally) | TR-12.1, TR-12.4 (S-5=A) | event-consumer derivation (D-4 structure) |
| INV-5 | **No quota restore while the linked reservation remains active** | TR-12.4 | entity guard change `[DEFERRED]` |
| INV-6 | **No double restore** — restore exactly once per business event | TR-11.4, TR-12.3 | `[DEFERRED]` |
| INV-7 | **One canonical pickup record**; consumption without a pickup record impossible; vouchers are authorization only | TR-4.9 (S-1=A) | — |
| INV-8 | **Availability never silently clamps business facts** — invalid rows flagged UNRESOLVED with provenance | TR-10.4, TR-10.5, S-2 §2 | existing flags preserved |
| INV-9 | **Overbooking is explicit only** — sole facility is transient `overbooking_limits`; contract overcommit forbidden | TR-10.5, TR-1.5 | — |
| INV-10 | **Wash idempotency** — repeat execution for the same range is a no-op | spec §32 `[SPEC-CARRIED]` + TR-11.4 | `[DEFERRED]` |
| INV-11 | **Attrition idempotency** — assessed exactly once, at the single wash event, inside one transaction | TR-6.7 | — |
| INV-12 | **Atomic counter/reservation/record changes** — partial commit prohibited in any failure mode | TR-4.2, TR-11.6, TR-9.5, TR-2.5 | shared tx vs saga `[DEFERRED]` (D-9) |
| INV-13 | **Deterministic invalid-state handling** — illegal transitions reject deterministically; conflicts surface `CONFLICT`; no silent last-write-wins | TR-11.2, TR-2.1 | `[DEFERRED]` |
| INV-14 | **Core inequalities after every commit:** `picked ≤ contracted − released` (block); `picked + released ≤ quota` (allotment) | TR-2.4, TR-3.2, TR-11.3 | `[DEFERRED]` |
| INV-15 | **No lost updates; version monotonicity** | TR-11.1, TR-11.5 | `[DEFERRED]` (TR-11.7) |
| INV-16 | **GUARANTEED_BLOCK never washes** | TR-6.3 (spec `:1146`) | type filter |
| INV-17 | **Stop sale never changes quantity or releases rooms**; evaluation order: stop sale before remaining | TR-7.2, TR-7.4, TR-4.5 | — |
| INV-18 | **Single availability authority** — no second sellable number anywhere (incl. rebuilt A3) | TR-1.2, TR-10.1, TR-15.5 | — |
| INV-19 | **Every published event has ≥1 consumer; event-less mutation prohibited** (block ops, wash) | D-5, TR-2.5, TR-6.5 | outbox→queue→handler (unchanged plumbing) |
| INV-20 | **Declared schema** — every executed table/column reproducible from version-controlled artifacts | TR-15.3, TR-15.4 (D-2=A) | forward migration mechanism family `[DECIDED]`, written in Implementation |
| INV-21 | **Identity never asserted from a name** — hotel-scoped exact matching only, conflict handled explicitly | TR-4.10, TR-9.6 (D-11=C amended) | caller contract `[DEFERRED]` |

`[SPEC-CARRIED]` constraints (binding only as design reference, **not** re-decided): attrition bounds 50–100%; net rate < BAR at contract creation; no overlapping stop sales for same date/category (or merged); `releaseDays > 0` for `ROLLING_RELEASE`; `depositPaid ≥ 0`; `masterCreditLimit > 0` (spec §20/§22).

---

## 15. Error / Edge Cases

Rule: **no new business rules are invented.** Where the resolved set + audit + spec are silent, the row says so.

| Case | Domain behavior | Source |
|---|---|---|
| **USED voucher cancelled, reservation active** | Quota **stays consumed**; voucher state → `CANCELLED` only when the reservation path permits (command delegates to reservation-cancel) | TR-12.4 |
| **USED voucher cancelled, reservation cancelled** | Quota restores (exactly once, atomic) | TR-12.4, TR-12.1 |
| **ISSUED voucher cancelled** | `picked--` restore (floor-bounded) | TR-12.5 |
| **Missing `reservationId` on consume** | Consume **creates** the reservation (D-10-B) — the path cannot legitimately end with a missing link; fabricated ids prohibited; field is real-id-or-null | TR-3.4, D-10 |
| **Anomalous `USED` row with null `reservationId`** | Defensive surfacing required; **no guessed quota rule** | D-10 note 4 — SOURCE-SILENT |
| **Reservation already cancelled** | Restore already applied by cascade; duplicate application → deterministic no-op | TR-11.4, TR-12.1 |
| **Reservation already checked in (voucher cancel)** | Handled as a **regular reservation cancellation** (reservation domain rules lead) | spec `:725` `[SPEC-CARRIED]`, TR-12.1 |
| **No-show** | Quota **not** restored (consumption stands; past release window) | spec `:728`/`:1116` `[SPEC-CARRIED]`; pickup record status — SOURCE-SILENT |
| **Checkout** | Pickup → `CHECKED_OUT`; counters unchanged; FO direct GBA writes removed | TR-4.8, TR-9.4, D-4 |
| **Duplicate pickup submission** | Counters exactly-once (TR-11.4); voucher path idempotent (same voucher → same reservation) | TR-11.4, spec §32; request-dedup mechanics `[DEFERRED]` |
| **Duplicate cancellation** | Idempotent no-op; counters restored at most once | TR-12.3, TR-11.4 |
| **Insufficient quota** | Rejected by `picked + released ≤ quota` guard — on **all** intake paths, one formula | TR-3.2, S-4 |
| **Quota > physical at create/set-quota** | Rejected (guard to be implemented) | TR-1.5, TR-3.5 |
| **Historical `quota > physical` rows** | Surfaced as flagged facts; **never rewritten** | TR-1.5, TR-10.4/10.5 |
| **Concurrent pickup** | Exactly one wins; loser fails with `CONFLICT` or retries; never merged | TR-11.1/11.2/11.3; mechanism `[DEFERRED]` |
| **Concurrent cancellation (vs pickup/restore)** | Exactly-once restore guaranteed as invariant; mechanism `[DEFERRED]` | TR-11.4/11.5 |
| **Concurrent wash** | Same invariant set; wash unit atomic | TR-11.6, D-9 |
| **Repeated wash** | No-op for same range | spec §32 `[SPEC-CARRIED]` |
| **Repeated manual release** | Guard `release ≤ held − picked` re-evaluated; counters never violate INV-14; request-dedup `[DEFERRED]` | TR-5.1, TR-11.3/11.4 |
| **Partial transaction failure** | Whole unit rolls back; error surfaces; **no** compensating-swallow (current `rollbackQuota` pattern rejected as a guarantee) | D-9, TR-11.6 |
| **Pickup on ineligible block state** | Rejected (`DRAFT`/`TENTATIVE`/`CLOSED`/`CANCELLED`) | TR-2.3 |
| **Shoulder removal with active pickups** | Rejected; rollups recomputed on allowed removal | TR-8.3 |
| **Stop sale active + intake** | Blocked on all allotment paths (before quantity check); display remaining 0 | TR-7.1, TR-7.4 |
| **Allocation removal with active pickups** | Rejected | TR-2.4 |

---

## 16. Event Contract (D-5)

**Mandatory:** every business-state change in this domain publishes exactly one domain event; **every published event has at least one consumer** (INV-19). Mechanism (outbox → queue → handler) is unchanged — `[DEFERRED]` for transport, decided only as: events exist, events are consumed, event-less mutation is prohibited (TR-2.5, TR-6.5, TR-11.6).

### 16.1 GBA domain events (producers)

`[DECIDED = the 7 Phase 3 events]; [SPEC-CARRIED = events implied by spec but not re-decided, published at implementation]`

| Event | Producer (context) | Must carry | Consumers → effect |
|---|---|---|---|
| `BlockCreated` | GBA (Reservations/Front Office invoke) | hotel_id, block_id, block_code, dates, policy, status | Availability: (re)compute eligibility |
| `BlockStatusChanged` | GBA | from/to status, hotel_id, affected dates | Availability: drop/add contribution (TR-10.3) |
| `PickupCreated` | GBA | hotel_id, reservation_id (real), block/allotment id, dates, room types, counters snapshot | Reservation domain: association truth; Availability: assert/publish |
| `PickupCancelled` | GBA | hotel_id, reservation_id, restore counts, actor | Reservation: unlinked; **Availability: restore + invalidate** (TR-12.6) |
| `VoucherIssued` | GBA | hotel_id, voucher_id, code, allotment id, dates | Availability: input changed |
| `VoucherConsumed` | GBA | hotel_id, voucher_id, **real reservation_id** | Reservation: link exists; pickup record created |
| `VoucherCancelled` | GBA | hotel_id, voucher_id, restore? (per TR-12.4) | Availability: invalidate if restored |
| `ManualReleaseExecuted` | GBA | hotel_id, block/allotment, qty, actor, before/after | Availability: released rooms rejoin sellable (TR-5.2) |
| `CutOffWashExecuted` | GBA | hotel_id, block_id, date, released qty, reason, per-category, before/after | Availability: rejoin (TR-6.5); attrition already inside same tx |
| `AllotmentWashExecuted` | GBA | hotel_id, allotment_id, window, released qty | Availability: rejoin (TR-6.5) |
| `ShoulderAllocationChanged` | GBA | hotel_id, block_id, dates, qty delta | Availability: invalidate (TR-8.4) |
| `StopSaleApplied` / `StopSaleLifted` | GBA | hotel_id, dates, category, reason | Selling permission fact (TR-7.x) — display only |
| `AttritionAssessed` | GBA | hotel_id, block_id, shortfall, liability, threshold | (read/report consumers; posting already in wash tx) |
| `[SPEC-CARRIED]` quota/batch-set events (spec §27 event list) | GBA | — | as required by "every event consumed" — enumerated at implementation |

### 16.2 Reservation events GBA consumes (D-4)

| Event | Producer | GBA consumer action (idempotent) |
|---|---|---|
| `reservation.cancelled` | Reservations | pickup → `CANCELLED`, unlink (TR-9.2), restore counters exactly once (TR-12.1/12.3), Availability invalidate (TR-12.6) |
| `reservation.checked_out` | Front Office | pickup → `CHECKED_OUT`; **no** counter change (TR-4.8) |
| **No ordering guarantee** across events — consumers must be order-tolerant (§7.2). Delivery: at-least-once assumed; failures retry via queue, never swallowed (D-4 confirmed rule) | | |

---

## 17. Data Model Contract

**Level of statement:** declared logical schema only — **NO SQL, NO index strategy, NO storage mechanics** (implementation family from D-2 deferred to Implementation Plan). Gaps are stated as gaps; **the absence of a migration is not permission to improvise.**

### 17.1 Declared new-world entities (exist in `packages/db/schema.prisma`, the D-2 decision baseline)

| Entity | Declared columns (key) | Notes |
|---|---|---|
| `group_bookings` | id, hotel_id, group_name, contact… | parent of blocks |
| `group_blocks` | id, hotel_id, group_booking_id, block_code (unique/hotel), arrival/departure_date, cutoff_date, block_type, inventory_policy, rate_plan_code, negotiated_rate, shoulder_days_before/after, status (default `DRAFT`), version, rollups | :17046+ |
| `group_block_allocations` | id, hotel_id, block_id, stay_date, room_type, contracted_qty, current_held_qty, picked_qty, released_qty, version | natural key (hotel, block, date, room_type) |
| `allotment_contracts` | id, hotel_id, allotment_code, type (triad), commitment (HARD/SOFT), release_days_before, validity_start/end, status, version, rollups | :17215 area |
| `allotment_quotas` | id, hotel_id, allotment_id, stay_date, room_type, quota, picked, released, stop_sale_active, net_rate, version | single guard formula lives here |
| `group_pickups` | id, hotel_id, reservation_id, block/allotment refs, status default `ACTIVE`, pickup_type, source | **the** association truth (TR-9.2) |
| `vouchers` | id, hotel_id, allotment_id, voucher_code (unique per hotel+allotment), status (`ISSUED` default), reservation_id nullable real, issued/expires dates | :17170 area |
| `stop_sales` | id, hotel_id, date range, category, status (`APPLIED`/`LIFTED`), reason | per hotel+date+category |
| `overbooking_limits` | hotel_id, room_type, date, allowance | **sole** overbooking facility (TR-10.5) |
| `wash_logs` / `release_logs` / `attrition` records | as declared | audit trail for §8.2/§10 |

**Declared gaps (D-2 decision: no new domain schema in Phase 4):** no `shoulder_days` table, no `allotment_pickups` (splits collapse into `group_pickups`), no `attrition_policies` table (policy columns on block), no separate `net_rate` history table. Debt absorbed by Phase 2b + 11 per AGENTS.md.

### 17.2 Legacy-retained entities (read-only, TR-15.1)

`reservations` (with GBA columns `block_code`/`allotment_code`…), legacy GBA tables (`group_bookings`/`group_blocks`-era rows in old naming), legacy `availability` counters (A4). **Never authoritative for any fact in this document** (TR-15.1, TR-15.5). GBA never writes legacy tables (TR-15.1).

### 17.3 Declared-then-executed (D-2 = A, TR-15.3/15.4)

Every table/column this domain depends on must be executable from version-controlled artifacts before Production. The exact forward mechanism family is `[DECIDED family]`, written up in the Implementation Plan — **not** specified here (mechanism-as-policy prohibition).

---

## 18. Legacy Boundaries

| Boundary | Rule | Source |
|---|---|---|
| Legacy GBA tables | **RETAIN**, read-only; excluded from writes | TR-15.1 |
| Legacy `availability` counters (A4/CRS engine) | **Never read, never written** by GBA paths — disjoint writer | TR-15.1, audit §3 |
| `reservations` GBA columns (`block_code` etc.) | Retained as **secondary read-optimization only**; pickup record is truth | TR-15.7, TR-9.2 |
| FO check-out direct GBA writes | **Removed**; event-driven via D-4 consumer + queue retry | D-4 confirmed rule (L-13) |
| Old calculator (`AllotmentDetail.tsx:189-212`) / `use-group-allotment.ts:305-313` cached totals / `AllocationStatus.tsx:146-149` 0.85 threshold | Replaced by Authority-derived values (TR-10.1/10.2); no local math | D-6 |
| `GET /group-bookings/available-rooms` (F-18) | Retired | D-6 |
| A3 Activities matrix | Rebuilt on Authority (D-6a = A); interim labeled view allowed **only** while labeled | D-6a |
| Dead `IInventoryCommitmentPort` / read-only assertion queries (A5) | Replaced by asserted path; wiring `[DEFERRED]` | D-7 note, TR-4.1 |
| A6 reads availability; GBA supplies the link — never mirrors logic | TR-4.1, TR-10.6 |
| Single source of truth: **pickup record**; single formula (TR-3.2); single availability authority (TR-10.1); single consumption ledger (TR-4.9) | TR-9.2, S-4, D-6, S-1 |
| Historical drift (F-1/F-2 quota-vs-picked; F-3 stop-sale; F-4 guessed counts; F-5 rendered-not-stored; F-6 swallowed rollback; F-7 unscoped SQL; F-8 event-less ops; F-9/%-threshold UI; F-10 stop-sale-loss; F-11 unguarded shoulder; F-12 code defect; F-13 A4 writes) | All are **non-conformant with this contract** and are addressed by its rules; migration/cleanup of drifted rows is **Phase 11 scope**, not re-decided here | audit 07 + this doc's supersession |
| Legacy migration/removal | Not started — Phase 11 (cutover rehearsal + rollback criteria) | AGENTS.md roadmap |

---

## 19. Reconciliation / Consistency Model

### 19.1 Read models

- **No stored read model exists yet** for the Availability side of GBA; all sellable figures derive on read from A1 inputs (TR-10.1) — consistency = input invalidation freshness (TR-10.3).
- Stale-by-design is prohibited: any figure that could be stale must carry provenance + `UNRESOLVED` flagging (TR-10.4) rather than render silently.
- Legacy rendered/cached figures (F-5) are non-authoritative and are replaced, not reconciled.

### 19.2 Reconciliation duties (read-side, no authority)

1. **Counters vs rows:** `picked` must equal the sum of ACTIVE + CHECKED_OUT pickup/voucher consumptions for that hotel/date/room type; `released` equals wash+manual release totals. Drift ⇒ `UNRESOLVED` flagged fact (TR-10.4), surfaced — never auto-rewritten (S-2 §2).
2. **Legacy vs new:** read-only comparison for migration readiness (Phase 11); legacy never wins (TR-15.5).
3. **Availability assertion totals:** A6/A5 compare published vs recomputed (existing assertion read-only queries) — discrepancies become flagged facts, not fixes (TR-10.4).
4. **Quota > physical rows:** flagged (TR-10.4), never silently rewritten (TR-1.5).
5. **Association truth:** pickup record ↔ reservation row must agree; mismatch surfaced as anomaly (D-10 note 4) — SOURCE-SILENT on repair semantics.

### 19.3 Failure / recovery posture

- Deterministic errors (`CONFLICT`, `INVALID_STATE`) are business signals — never retried into success (TR-11.2).
- Event-delivery failures retry via queue with idempotent handlers (D-4, TR-11.4); swallowed failures prohibited (TR-11.6 partial-failure rule).
- Compensating manual scripts are not a substitute for atomicity (D-9).
- Scheduler absence (TR-6.6 deferred) ⇒ washs that do not fire are **visible** (metadata marked non-authoritative, TR-6.5) — never silently "assumed done".

---

## 20. Traceability and Supersession

### 20.1 Decision → rule traceability (all 22 resolved items; 100% represented)

| Decision | Status (confirmed 2026-09-30) | Rules/sections implemented | Primary section(s) |
|---|---|---|---|
| D-1 pickup-new-world | A | TR-15.1/15.2/3.1/2.7 | §17.1, §18, §4, §5 |
| D-2 declared schema | A | TR-15.3/15.4/8.5, 2.7 | §17.1/17.3, §9 |
| D-3 cut-off/wash semantics | A | TR-5.5/6.1/6.2/6.3/6.4/6.5/6.6(def)/6.3 | §8.1/8.2, §4.5/4.6, §13.1 |
| D-4 reservation event lifecycle | B structure + C semantics | TR-4.8/9.4/15.6/12.1 | §7.2, §13.3, §16.2 |
| D-5 event contract | — (contract) | TR-6.5, INV-19 | §16, §19.3 |
| D-6 availability authority | (in A6 note) | TR-1.1/1.2/1.3/10.1/10.2/10.4/15.5 | §11.1/11.2, §18 |
| D-6a A3 rebuild | A | TR-15.5 (retire/flag), TR-10.1 | §11.1, §18 |
| D-7 pickup reservation creation | — (two-layer) | TR-1.4/4.1/9.1/9.7/10.6/12.6 | §7.1, §11.2 |
| D-8 concurrency | (conditional-version vs lock) | TR-11.1–11.7(def)/2.4/3.2 | §12.1/12.2 |
| D-9 atomicity | (tx vs saga) | TR-4.2/11.6/9.5 | §12.2, §14 INV-12 |
| D-10 voucher→real reservation | B | TR-3.4/4.4/9.3 | §6.1/6.3, §7.1, §15 |
| D-11 guest identity order | C amended | TR-4.10/9.6 | §7.1, §14 INV-21 |
| D-12 release verb | — (POST) | TR-5.4 | §8.3 |
| D-13 shoulder uniform | A | TR-8.1–8.6(c)/4.7 | §9, §4.3 |
| D-14 attrition | A | TR-6.7/6.8/6.9 | §10 |
| D-15 block lifecycle | — (first-class DRAFT→TENTATIVE) | TR-2.1–2.3/2.6/13.5/3.3 | §13.1, §4.4 |
| S-1 one pickup record | A | TR-4.9/3.4 | §6.1, §14 INV-7 |
| S-2 quota ≤ physical | A | TR-1.5/3.5/10.5 | §5.5, §14 INV-2/9 |
| S-3 contract-type mapping | A | TR-1.6/3.6 | §2.2, §5.3 |
| S-4 stop-sale scope | A (allotments only) | TR-4.5/7.1–7.5 | §5.6, §13.5, §15 |
| S-5 restore-iff-cancelled | A | TR-12.4 | §6.4, §13.2, §14 INV-4/5 |
| S-6 hotel isolation | A | TR-14.1–14.5 | §14 INV-1, §12.3, §18 |

**Deferred-only rules (not converted to decided rules anywhere in this document):** TR-6.6 (scheduler technology) and TR-11.7 (lock mechanism) — both marked `[DEFERRED]`, **IMPLEMENTATION-PLAN DEFERRED**, in §8.2, §12.1, §13.x. No other TR remains undecided: 97 = 95 DECIDED · 0 RECOMMENDED · 0 PENDING · 2 DEFERRED.

### 20.2 Supersession list (spec/audit items this document supersedes; decision wins, never silent)

| # | Superseded item | Superseded by |
|---|---|---|
| 1 | Spec block lifecycle starting at `TENTATIVE` / no `DRAFT` (gba `:1060-1070` area) | D-15: `DRAFT → TENTATIVE` first-class (§13.1) |
| 2 | Voucher spec stages `INTAKE→CONFIRMED→CHECKED_IN/CHECKED_OUT/NO_SHOW` (`:710-714`) | Stored voucher states `ISSUED/USED/CANCELLED/EXPIRED`; stages map to linked reservation + pickup record (§6.3) |
| 3 | Spec remaining formula ordering `:437` (remaining defined before stop-sale check) | TR-7.4 stop-sale checked **first** (TR-4.5), then `quota − picked − released` (§6.1, §5.2) |
| 4 | VO attrition default 85% / `per_day_minimum` / `cumulative` / `room_type_specific` (VO `:560-566` variants) | D-14=A: single 80% uniform threshold (§10) |
| 5 | Spec open question "overbooking?" (`:1714+`) left open | S-2=A closed: contract overcommit **prohibited**; transient limits sole facility (§5.5) |
| 6 | Spec stop-sale applying generally / ambiguous scope | S-4=A: allotment contracts only (§13.5) |
| 7 | Spec §39 open decision 1: association entity (reservation block_code as truth) | TR-9.2/15.7: pickup record is truth (§3.1, §17.2) |
| 8 | Spec term `Delegate` | `Pickup` (TR-1.6 mapping) — canonical vocabulary §2.1 |
| 9 | Spec `ON_HOLD` / `TERMINATED` contract states | `SUSPENDED` / `CLOSED` per mapping (§13.4) |
| 10 | Spec ledger ownership open question 2 | Marked `[DEFERRED]` implementation scope (§10) — not asserted either way |

Where the spec/audit is **silent** and no decision exists: marked `SOURCE-SILENT` with no rule invented (§7.3, §13.3, §6.4, §15).

---

## 21. Quality Gate

### 21.1 Required-content checks (17)

| # | Check | Result |
|---|---|---|
| 1 | All 22 decisions represented | ✅ §20.1 (22/22 rows) |
| 2 | All 6 confirmations represented (S-2, S-5, D-6a, D-10, S-1, S-3; 2026-09-30) | ✅ §20.1 + §5.5, §6.4, §11.1, §6.1, §2.2 |
| 3 | S-2 quota ≤ physical, violations flagged not rewritten | ✅ §5.5, INV-2 |
| 4 | S-5 restore-iff-reservation-cancelled | ✅ §6.4, INV-4/5, §13.2 |
| 5 | D-3 GUARANTEED_BLOCK never washes | ✅ §8.2, INV-16 |
| 6 | D-13 shoulder uniform (P1/P2/P3) | ✅ §9 |
| 7 | D-14 attrition at wash only, single tx | ✅ §10, INV-11 |
| 8 | D-11 name-matching prohibited, conflict handling | ✅ §7.1, INV-21 |
| 9 | S-1 one canonical pickup record both paths | ✅ §6.1, INV-7 |
| 10 | D-10 real reservation id, fabricated ids prohibited | ✅ §6.1, §7.1, §15 |
| 11 | Legacy never authoritative (TR-15.1/15.2/15.5/15.7) | ✅ §3.1, §17.2, §18 |
| 12 | Hotel isolation on every operation | ✅ INV-1, §12.3 |
| 13 | Deferred items (TR-6.6, TR-11.7) marked deferred, not converted | ✅ §8.2, §12.1, §20.1 |
| 14 | No implementation detail stated as policy (no lock syntax, no SQL, no index strategy) | ✅ §12, §17 |
| 15 | No schema change asserted (declared-schema gap language only) | ✅ §17.1/17.3 |
| 16 | Every section of the required 20-part structure present | ✅ §1–§20 |
| 17 | Quality gate itself present | ✅ §21 |

### 21.2 Contradiction pass (second sweep)

- Counter-timing table (§6.2) vs voucher table (§6.3) vs pickup table (§13.3): consistent — issue increments, consume doesn't, checkout never, cancel restores conditionally.
- Wash released-qty semantics (§8.2) vs remaining formula (§5.2) vs A1 inputs (§11.2): consistent — post-wash remainder reads 0, `picked` preserved.
- S-2 guard (§5.5) vs A1 `Math.max(0,·)` clamp (§11.2): consistent — clamp never becomes the enforcer because contract writes can't create negative state.
- D-4 consumer (§16.2) vs S-5 (§6.4): consistent — reservation-cancel is the sole restore authority; voucher-cancellation delegated to it.
- Scope/stop-sale (§13.5) vs pickup guard order (§6.1): consistent — stop sale before remaining, allotment paths only.
- D-6a (§11.1) vs §18 A3 row: consistent — same interim-label condition.
- No unresolved contradictions: **0**.

### 21.3 Declaration

This document is **FINAL** and **READY FOR IMPLEMENTATION PLAN**.
Content-only document: **no code, schema, migration, API, or test changes were made.**






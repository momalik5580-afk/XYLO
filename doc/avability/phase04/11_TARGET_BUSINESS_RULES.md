# Phase 4 — Target Business Rules (GBA / Allotment)

**Phase:** 4 — Decisions only. **TARGET rules only — no current-state comparison** (current state lives in `05_BUSINESS_RULES_CURRENT_STATE.md`; resolution + evidence live in `10_DECISION_RESOLUTION.md`).
**Date:** 2026-09-30

**Status tags on every rule:**
`[DECIDED]` = evidence-forced, locked · `[RECOMMENDED]` = pending user confirmation (see `12_DECISION_STATUS.md` §2) · `[PENDING USER]` = business choice required before implementation · `[DEFERRED]` = explicitly later-phase (mechanism or scope)

**Sources:** `D-x` = `10_DECISION_RESOLUTION.md` · `T-x` = spec rule registry (`05_…` §2, with spec line numbers) · `F-x` = findings (`09_…` §2).
**Cross-cutting preamble (inherited by every rule below):** all rules are **hotel-scoped** per §14, and all state-changing rules obey §11 (concurrency invariants). No schema, mechanism, or implementation syntax is specified anywhere in this document.

---

## 1. Inventory (facts, capacity, policy)

- **TR-1.1** `[DECIDED]` (D-6) Exactly four fact kinds exist, tracked separately: **physical quantity** (rooms the hotel has), **selling permission** (may this date/category be sold — stop sale, restrictions), **reservation commitment** (rooms consumed by reservations), and **history** (wash/release/audit records). Only their combination into *sellable availability* is computed — and only by the single authority (§10).
- **TR-1.2** `[DECIDED]` (D-6, T-3) **Availability is the sole source of the sellable number.** No second inventory engine exists; GBA never publishes its own availability figure.
- **TR-1.3** `[DECIDED]` (D-6) **Block-held capacity is an inventory fact, not availability policy:** a `DEFINITE`/`OPEN_FOR_PICKUP` DEDUCT block's contracted quantities reduce sellable capacity; how they reduce it is defined solely by §10.
- **TR-1.4** `[DECIDED]` (D-7) A **pool** (one block's or one allotment's held rooms) and **transient sellable capacity** are distinct quantities. A pickup consumes from its pool *and* must be validated against transient capacity (two-layer check, never either alone).
- **TR-1.5** `[PENDING USER]` (S-2) **Allotment quota ≤ physical inventory** (no overbooking) — *recommended, awaiting decision.* If overbooking is adopted, it must appear as an explicit flagged fact, never as silent clamping.
- **TR-1.6** `[DECIDED]` (S-3) Contract-type references in all rules use the **canonical vocabulary** `ROLLING_RELEASE | GUARANTEED_BLOCK | FREE_SALE` (+ soft/hard commitment semantics); stored legacy values map through a published translation. `[RECOMMENDED]` on the canonical direction (spec vs code) — see `12_…`.

---

## 2. Blocks (group block lifecycle, allocations)

- **TR-2.1** `[DECIDED]` (D-15, T-6) Block status changes **only through explicit lifecycle transitions**: `DRAFT → TENTATIVE → DEFINITE → OPEN_FOR_PICKUP → CLOSED`, with `→ CANCELLED` permitted from any pre-closed state. The `DRAFT→TENTATIVE` step is first-class. Implicit/automatic status change is prohibited; widening availability eligibility to compensate is prohibited.
- **TR-2.2** `[DECIDED]` (D-15) **Only `DEFINITE` and `OPEN_FOR_PICKUP` blocks hold inventory.** `DRAFT`/`TENTATIVE` hold nothing; `CLOSED`/`CANCELLED` hold nothing.
- **TR-2.3** `[DECIDED]` (D-15) **Pickup requires an eligible block state** (`OPEN_FOR_PICKUP`; minimum acceptable `DEFINITE` per implementation choice): `DRAFT`, `TENTATIVE`, `CLOSED`, `CANCELLED` reject pickups.
- **TR-2.4** `[DECIDED]` (D-8 invariant 3; existing guards preserved) Per `hotel_id` + stay date + room type: **`picked ≤ contracted − released`** after every committed operation. Reducing a daily allocation below its picked quantity is rejected; removing an allocation with active pickups is rejected.
- **TR-2.5** `[DECIDED]` (D-13 base) **Every block mutation** (create/set/remove allocation, shoulder add/remove, release, wash, pickup, status change) **runs through guarded aggregate logic, emits its domain event, and commits atomically** (§11). Raw, unguarded, event-less mutation of block state is prohibited.
- **TR-2.6** `[DECIDED]` (D-15) Each block belongs to exactly one group booking and one hotel; its identity (`block_code`) is unique per hotel and generated correctly (defect F-12 fixed as part of implementation — behavior: code derived from actual count, not from a non-matching UUID-column query).
- **TR-2.7** `[DECIDED]` (D-1, T-4) The new-world tables are authoritative for blocks; pickup containment invariants (dates within block range, category within block categories, rate consistency — T-4 `:1128-1136`) are validated at pickup (§4).

---

## 3. Allotments (contracts, quotas, vouchers)

- **TR-3.1** `[DECIDED]` (D-1) The authoritative allotment entities are the new-world contract + daily quota tables; legacy `allotment` is read-only history.
- **TR-3.2** `[DECIDED]` (S-4 guard formula, D-8) Per `hotel_id` + date + room type: **`picked + released ≤ quota`** after every committed operation; consumption uses remaining = `quota − picked − released` — one formula for all intake paths.
- **TR-3.3** `[DECIDED]` (existing rule preserved; D-15 analog) An allotment accepts consumption only while `ACTIVE` (and not expired — §13). Contract states follow the lifecycle `DRAFT → ACTIVE → SUSPENDED/CLOSED/EXPIRED` with `SUSPENDED ⇄ ACTIVE` (spec T-6 allows `ON_HOLD` naming; mapping per TR-1.6).
- **TR-3.4** `[DECIDED]` (D-10 rule, S-1) **Every consumption of an allotment results in a pickup record linked to a real reservation.** Fabricated identifiers are prohibited; linkage is verifiable.
- **TR-3.5** `[PENDING USER]` (S-2) Overbooking policy (TR-1.5) applies here: quota is capped at physical unless the user decides otherwise.
- **TR-3.6** `[DECIDED]` (S-3) Wash/roll eligibility keys off **canonical contract type** (e.g., `GUARANTEED_BLOCK` never washes — §6); stop-sale applicability keys off contract presence (§7), never on group blocks (T-8).

---

## 4. Pickup (the consumption operation)

- **TR-4.1** `[DECIDED]` (D-7) **Two-layer validation:** (a) pool layer — capacity exists for every stay date + room type in range; (b) reservation layer — the created reservation passes the **same** availability/assertion lifecycle as any other reservation. Neither layer may be skipped, on any path, under any policy.
- **TR-4.2** `[DECIDED]` (D-9) **Pickup is atomic:** reservation row + pickup record + counter increments (+ folio effects where part of the operation) commit together or not at all. A reservation without its counter increment — or the reverse — is a prohibited state in every failure mode. Compensating scripts are not a substitute for atomicity.
- **TR-4.3** `[DECIDED]` (D-8) Concurrent pickups serialize per the §11 invariants: no lost increments, conflicts rejected/retried, never merged.
- **TR-4.4** `[DECIDED]` (D-10) Pickup records reference **only real reservation ids**; never synthetic tokens.
- **TR-4.5** `[DECIDED]` (S-4) Pickup validates **stop sale** (allotment pickups; blocks are out of scope per §7) and remaining-per-TR-3.2, in that order.
- **TR-4.6** `[DECIDED]` (T-4) Pickup respects date containment (check-in ≥ block/contract start; check-out ≤ end), category containment, and rate consistency; violations reject the pickup.
- **TR-4.7** `[DECIDED]` (D-13 base) Shoulder-date pickups follow the identical rules (guards, atomicity, events) as core-date pickups.
- **TR-4.8** `[DECIDED]` (D-4) **Counter truth:** counters increment exactly once per consumption event and decrement only when the consumption is undone (reservation cancelled / pickup cancelled) — checkout does **not** decrement (the room was consumed).
- **TR-4.9** `[RECOMMENDED]` (S-1) One canonical pickup record type for both entry paths (voucher-driven and direct); vouchers are authorization artifacts, pickup records are the ledger. *(Pending user confirmation.)*
- **TR-4.10** `[PENDING USER]` (D-11) Guest resolution never merges by name; explicit id → hotel-scoped email → create new. *(Pending decision.)*

---

## 5. Release (manual return of held rooms)

- **TR-5.1** `[DECIDED]` (existing guard preserved, corrected) A manual release may return only **unpicked** held rooms: release quantity ≤ `held − picked` (the current `canRelease ≤ held` allowance can push `picked > held` and is prohibited). Releasing picked rooms is not a release — it requires cancelling the consuming reservation (§12).
- **TR-5.2** `[DECIDED]` (D-6, D-9) Released rooms return to sellable inventory **through Availability** (its inputs invalidate immediately per §10/D-5 events); release commits atomically and emits its domain event.
- **TR-5.3** `[DECIDED]` (D-15/D-13 guards) Release is impossible against `DRAFT`/`CLOSED`/`CANCELLED` blocks and non-`ACTIVE` allotments; per-hotel scoping applies (§14).
- **TR-5.4** `[DECIDED]` (D-12) The release command's canonical API verb is `POST`.
- **TR-5.5** `[PENDING USER]` (D-3) Whether manual release **coexists** with scheduled wash depends on the D-3 decision (recommended: yes — manual remains as operator override alongside automation).

---

## 6. Rolling Release / Cut-off Wash (time-based return)

- **TR-6.1** `[PENDING USER]` (D-3, T-1, T-2 — **recommend adopt**) Cut-off wash: at the cut-off date, unpicked allocated rooms of a block return to general inventory in a **single wash event per block**, logged with date, count, reason.
- **TR-6.2** `[PENDING USER]` (D-3, T-2 — **recommend adopt**) Rolling release: allotment rooms wash back automatically **N days before arrival** for `ROLLING_RELEASE` contracts if not picked.
- **TR-6.3** `[DECIDED]` (T-2, spec `:1146` — invariant regardless of D-3 outcome) **`GUARANTEED_BLOCK` never washes** — the hotel is entitled to 100% regardless of pickup; wash mechanisms must exclude it by type.
- **TR-6.4** `[PENDING USER]` (D-3 sub-question — **recommend date granularity**) Wash timing is **date-granularity** (columns are dates): a wash applies from the start of the applicable day, not an arbitrary clock time.
- **TR-6.5** `[DECIDED]` (D-5, D-13) A wash mutates counters atomically, emits its wash event (`CutOffWashExecuted`/`AllotmentWashExecuted`), and invalidates Availability — event-less wash (or no wash at all while fields claim otherwise) is prohibited *once D-3 is decided*; until then cutoff/release fields are **non-authoritative** (must not be presented as active behavior).
- **TR-6.6** `[DEFERRED]` (D-3 mechanism) Scheduler technology (platform job pattern) is implementation scope.

---

## 7. Stop Sale (selling permission)

- **TR-7.1** `[DECIDED]` (S-4, T-8, spec `:628-646`) Stop sale is a **selling-permission** fact on **allotment contracts only**. While active for date + category: **all** allotment intake paths (voucher issue **and** direct pickup) are blocked. Group-block pickups are never stop-sale restricted.
- **TR-7.2** `[DECIDED]` (S-4) Stop sale **never changes quantity and never releases rooms**: physical quantity, `picked`, `released`, and `contracted` are untouched by stop-sale activation or lift. Reported "remaining" under stop sale reads as 0 **for selling display only** (spec `:437`) — the underlying counters are unchanged.
- **TR-7.3** `[DECIDED]` (S-4) Lifting a stop sale restores **selling only** — it grants back no inventory that was never removed.
- **TR-7.4** `[DECIDED]` (existing guard preserved) Stop-sale evaluation occurs before remaining-quantity evaluation on every intake path (one order, one formula).
- **TR-7.5** `[DECIDED]` (S-6) Stop-sale state is per hotel + date + category, read/written hotel-scoped.

---

## 8. Shoulder Days (pre/post block dates)

- **TR-8.1** `[DECIDED]` (D-13 base) Shoulder days are **ordinary daily allocations within the extended block date range** — same entities, same guards, same events, same atomicity as core-day allocations.
- **TR-8.2** `[DECIDED]` (D-13 base) Shoulder allocations are created with **declared quantities** (silent qty-0 placeholders prohibited) and the shoulder counters accumulate correctly (overwriting the opposite direction prohibited).
- **TR-8.3** `[DECIDED]` (D-13 base, F-11) Removing shoulder days obeys the **no-removal-with-active-pickups** guard and recomputes block rollups; unguarded deletion is prohibited.
- **TR-8.4** `[DECIDED]` (D-13 base) Shoulder mutations emit domain events and invalidate Availability like any allocation change.
- **TR-8.5** `[DECIDED]` (D-2 rides) The shoulder bookkeeping (before/after counts) is declared schema — reproducible in a fresh environment.
- **TR-8.6** `[PENDING USER]` (D-13 differentiators — **recommend uniform: Y/Y/Y**) Three policy questions remain: (a) do shoulder rooms count toward the attrition base? (b) do cut-off/wash rules apply to them exactly as core days? (c) are pickups on shoulder dates unrestricted? *Recommended answer: yes to all three (uniform treatment).*

---

## 9. Reservations (pickup-created & associated)

- **TR-9.1** `[DECIDED]` (D-7) A reservation created by pickup is a **first-class reservation**: same lifecycle, same assertion/availability path, same status rules as any other reservation. `source`/`pickup_type` may label its origin but never its rules.
- **TR-9.2** `[DECIDED]` (T-9, D-10) The **pickup record is the authoritative association** between a reservation and its block/allotment. Bare linkage columns on `reservations` (group/allotment columns) are secondary/denormalized during transition; removal of the columns is a Phase 11 migration decision (not made here).
- **TR-9.3** `[DECIDED]` (D-10) Every reservation↔voucher/block association stores a **real reservation id** or is null; synthetic ids are prohibited.
- **TR-9.4** `[DECIDED]` (D-4) Reservation lifecycle drives pickup state: `checked_out` → pickup marked consumed (counters unchanged); `cancelled` → pickup cancelled **and counters restored**. FO checkout performs **no direct writes** to GBA tables (L-13 closed) — events carry the state (D-5).
- **TR-9.5** `[DECIDED]` (D-9) Reservation creation, pickup record, and GBA counters commit atomically; partial states are prohibited.
- **TR-9.6** `[PENDING USER]` (D-11) Guest linkage of pickup-created reservations follows TR-4.10 once decided.
- **TR-9.7** `[DECIDED]` (D-7) Status of a pickup-created reservation is **not** hardcoded past the assertion outcome; creation honors the same confirmation semantics as the reservation domain (mechanism `DEFERRED`).

---

## 10. Availability (authority & invalidation)

- **TR-10.1** `[DECIDED]` (D-6) **Single authority:** the Availability computation (A1 — snapshot/assertion path) is the only producer of sellable availability. The Activities matrix (A3) and `available-rooms`-style endpoints derive from it or are retired; independent eligibility filters elsewhere are prohibited (F-4, F-18 closed by this rule).
- **TR-10.2** `[DECIDED]` (D-6) Eligibility filters live **in one place**: DEDUCT-policy evaluation, booking-status set, soft-deleted exclusion, block-status whitelist (`DEFINITE`/`OPEN_FOR_PICKUP`), contract-type + validity-window checks are defined once and consumed everywhere.
- **TR-10.3** `[DECIDED]` (D-5, D-7) Availability **invalidates on every GBA mutation** that changes held/picked/released quantities (block create/confirm/open/release/wash/cancel; allotment quota/release/wash/stop-sale lift where selling permission changes) — delivered via event consumers (D-5). Stale-by-design is prohibited.
- **TR-10.4** `[DECIDED]` (D-6) Availability reports **provenance** (which facts fed the number) and flags unresolved facts — the existing source-flagging behavior is part of the contract.
- **TR-10.5** `[DECIDED]` (D-6, S-2) If overbooking (TR-1.5) is ever adopted, the overcommit appears as an **explicit flagged fact** inside the authority — never a second number or silent clamp.
- **TR-10.6** `[DECIDED]` (D-7) Pickup queries/consults this authority (T-3's duty); GBA does not fork its own calculation (dead `IInventoryCommitmentPort` pattern replaced by a wired port — mechanism `DEFERRED`).

---

## 11. Concurrency & Transaction Invariants (business level only)

- **TR-11.1** `[DECIDED]` (D-8.1) **No lost updates:** concurrent writes to counters never merge; exactly one wins, the other fails or retries.
- **TR-11.2** `[DECIDED]` (D-8.2) **Conflict is an error:** a conflicting write surfaces `CONFLICT` to the caller; silent last-write-wins is prohibited.
- **TR-11.3** `[DECIDED]` (D-8.3) **Core inequality holds after every commit:** block `picked ≤ contracted − released`; allotment `picked + released ≤ quota` (per hotel/date/category).
- **TR-11.4** `[DECIDED]` (D-8.4) **Exactly-once counter mutation per business event:** retries are idempotent; double-application is prohibited.
- **TR-11.5** `[DECIDED]` (D-8.5) **Version monotonicity:** staleness detection never rolls backwards; writes from stale state never commit.
- **TR-11.6** `[DECIDED]` (D-9) **Atomicity:** pickup (§4), release (§5), wash (§6), shoulder changes (§8), and status transitions (§2/§3) are each one atomic unit — multi-step partial commit prohibited.
- **TR-11.7** `[DEFERRED]` (D-8 mechanism) The concrete mechanism (conditional-version retry vs load-time lock) is implementation plan scope — this document states no locking syntax.

---

## 12. Cancellation

- **TR-12.1** `[DECIDED]` (D-4, S-5 recommendation basis) **Cancelling the consuming reservation cancels its pickup and restores counters** (quota/`picked` returned) — the reservation is the source of truth for whether consumption stands.
- **TR-12.2** `[DECIDED]` (existing rules preserved) Cancelling a block/allotment: blocks → `CANCELLED` via lifecycle (TR-2.1), releasing held rooms to inventory through Availability; allotments → `CLOSED` per TR-3.3. Soft-deletion preserves history; hard deletion of committed history is prohibited.
- **TR-12.3** `[DECIDED]` (S-1/§4) Cancelling a pickup record itself restores counters exactly once (idempotent with TR-11.4) and unlinks its reservation association per TR-9.2.
- **TR-12.4** `[PENDING USER]` (S-5 — **recommend follow reservation**) Cancelling a **USED** voucher: quota restores **iff** the linked reservation is cancelled (follow reservation lifecycle) — never "always" (restores under a live reservation → over-sell) and never "never" (burns quota permanently). *Pending user decision.*
- **TR-12.5** `[DECIDED]` (existing behavior preserved) Cancelling an **ISSUED** (unconsumed) voucher restores its held quota.
- **TR-12.6** `[DECIDED]` (D-7) Cancellation flows through the same authority: restored capacity invalidates Availability (TR-10.3) atomically with the counter restore.

---

## 13. Expiry (time-driven terminal states)

- **TR-13.1** `[DECIDED]` (TR-3.3, existing `assertNotExpired`) An allotment past `validity_end` stops accepting consumption and transitions to `EXPIRED` — expiry blocks intake, it does not by itself wash/destroy data.
- **TR-13.2** `[DECIDED]` (D-6/A1 rules preserved) Expired/closed contracts contribute **zero** commitment via the validity-window filter in the single authority (TR-10.2).
- **TR-13.3** `[DECIDED]` (existing voucher rules preserved) Vouchers reach `EXPIRED` per their lifecycle (`ISSUED → EXPIRED`); expired vouchers cannot be consumed or (per TR-12.5 analog) restore nothing beyond issued-state handling.
- **TR-13.4** `[PENDING USER]` (D-3) Whether expiry **triggers wash** of unconsumed quota (vs leaving it to manual release) follows the D-3 scheduled-vs-manual decision; GUARANTEED never washes either way (TR-6.3).
- **TR-13.5** `[DECIDED]` (D-15) Block expiry-of-relevance is expressed through lifecycle (`CLOSED`) and wash, not through silent data aging; `CLOSED` blocks hold nothing (TR-2.2).

---

## 14. Hotel Isolation (multi-tenant)

- **TR-14.1** `[DECIDED]` (S-6) **Every read and write in this domain — including raw SQL — carries `hotel_id` in its predicate.** Mutation by bare id is prohibited.
- **TR-14.2** `[DECIDED]` (S-6) The `x-property-id` header override is permitted **only for an explicitly privileged role**; it is never silently effective in production. Development access bypasses must never be active where real tenant data exists.
- **TR-14.3** `[DECIDED]` (S-6) Cross-hotel operations (e.g., transfer intents) require explicit multi-property authorization that does not yet exist — all Phase 4 rules are single-hotel.
- **TR-14.4** `[DECIDED]` (S-6) Hotel scoping applies to the *inputs* of the single availability authority: no availability computation may aggregate across hotels (TR-10.2 filters are per-hotel by construction).
- **TR-14.5** `[DECIDED]` (S-6) Every rule in §1–§13 above is per-hotel; natural keys and identity uniqueness (TR-2.6, TR-3.2, TR-7.5) are hotel-qualified.

---

## 15. Legacy Transition (GBA legacy → new world)

- **TR-15.1** `[DECIDED]` (D-1) The new GBA world is **authoritative**. Legacy GBA tables (`allotment`, `block_*`) are `RETAIN` — read-only for historical reporting — until Phase 11; **no new code writes them**.
- **TR-15.2** `[DECIDED]` (D-1) Removal/reclassification of legacy tables is a **Phase 11** decision (`REMOVE AFTER CUTOVER` candidates), never an incidental side effect of feature work.
- **TR-15.3** `[RECOMMENDED]` (D-2) **Declared-schema rule:** every table/column executed by runtime GBA code is reproducible from version-controlled schema artifacts (forward migration after live verification — *pending confirmation*).
- **TR-15.4** `[DECIDED]` (D-2 domain requirement) Code may not depend on objects that exist only in one live database; shadow artifacts are either declared or removed (removal requires the behavior decision D-2 defers).
- **TR-15.5** `[DECIDED]` (D-6) During transition, old and new availability computations do **not** coexist: derived/relabelled views only (D-6a path pending confirmation).
- **TR-15.6** `[DECIDED]` (D-4) Legacy-verdict statuses ("KEEP until Phase 4") expire with this phase's decisions: FO direct writes to GBA tables end when D-4's recommended path is implemented.
- **TR-15.7** `[DECIDED]` (T-9, TR-9.2) Dual association patterns (columns + pickup records) are tolerated **only** as read-optimization during transition; the pickup record is the truth (columns removal deferred to Phase 11).

---

## Rule Count & Status Summary

| Section | Rules | DECIDED | RECOMMENDED | PENDING USER | DEFERRED |
|---|---|---|---|---|---|
| 1 Inventory | 6 | 4 | 1 (S-3 direction) | 1 | 0 |
| 2 Blocks | 7 | 7 | 0 | 0 | 0 |
| 3 Allotments | 6 | 5 | 0 | 1 | 0 |
| 4 Pickup | 10 | 8 | 1 | 1 | 0 |
| 5 Release | 5 | 4 | 0 | 1 | 0 |
| 6 Rolling/Wash | 6 | 2 | 0 | 3 | 1 |
| 7 Stop Sale | 5 | 5 | 0 | 0 | 0 |
| 8 Shoulder | 6 | 5 | 0 | 1 | 0 |
| 9 Reservations | 7 | 6 | 0 | 1 | 0 |
| 10 Availability | 6 | 6 | 0 | 0 | 0 |
| 11 Concurrency | 7 | 6 | 0 | 0 | 1 |
| 12 Cancellation | 6 | 5 | 0 | 1 | 0 |
| 13 Expiry | 5 | 4 | 0 | 1 | 0 |
| 14 Hotel Isolation | 5 | 5 | 0 | 0 | 0 |
| 15 Legacy Transition | 7 | 6 | 1 | 0 | 0 |
| **Total** | **94** | **78** | **3** | **11** | **2** |

*PENDING USER rules map 1:1 to the items in `12_DECISION_STATUS.md` §2. No rule in this document may be implemented while its PENDING USER flag is unresolved, except rules explicitly tagged `[DECIDED]` alone.*

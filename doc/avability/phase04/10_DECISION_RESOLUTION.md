# Phase 4 — Decision Resolution (D-1…D-15 + Supplementary)

**Phase:** 4 — GBA / Allotment Integration (Decisions only — NOT implementation)
**Date:** 2026-09-30
**Inputs:** `08_OPEN_DECISIONS.md` (D-1…D-15), `09_PHASE_4_AUDIT_SUMMARY.md` (F-1…F-19), `05_BUSINESS_RULES_CURRENT_STATE.md` (T-1…T-11, B-1…B-13), `04_AVAILABILITY_INTEGRATION_MATRIX.md` (A1…A9), `06_CONCURRENCY_AND_TRANSACTION_AUDIT.md` (C-1…C-17), `docs/design/gba-domain-spec.md`.
**Scope honored:** 0 code, 0 schema, 0 migration, 0 backfill, 0 API/DTO/controller/handler/repository changes, 0 tests implemented, 0 implementation plan. Targeted source reads were performed only to validate specific decisions (each cited below).

---

## 0. Method & Conventions

**Decision statuses used in this document:**

| Status | Meaning |
|---|---|
| `DECIDED` | Evidence (codebase state or spec requirement) forces a single answer; no reasonable business choice remains |
| `RECOMMENDED — USER CONFIRMATION REQUIRED` | Evidence narrows to one best option, but the choice is policy, not fact → needs one confirmation |
| `USER DECISION REQUIRED` | Two or more defensible business policies exist; evidence cannot pick one → user must choose (recommendation given) |
| `DEFERRED` | Genuinely belongs to a later phase or the implementation plan; recorded so it is not lost |

**Impact dimensions** (only relevant dimensions are discussed per decision, from the 18):
`Inventory/Availability` · `Reservation lifecycle` · `Guest data` · `Billing/Folio` · `Reporting/Analytics` · `Front-office ops` · `Concurrency/Integrity` · `Multi-tenant isolation` · `UX/Operations` · `Migration/Cutover risk` · `Implementation complexity` · `Testability` · `Rollback safety` · `Spec conformance` · `Data honesty (stored-but-unenforced)` · `Performance` · `Security/Permissions` · `Dependency ordering`

**Evidence rule:** every Current-state statement cites `file:line`. Spec citations use the T-number registry from `05_BUSINESS_RULES_CURRENT_STATE.md` §2 (which carries spec line numbers).

---

## 1. Decision Dependency Graph

Decisions are resolved in dependency order (NOT alphabetical):

| # | Decision | Depends on | Why it sits here |
|---|---|---|---|
| 1 | **D-1** authoritative data world | — | Foundation for every schema/legacy statement |
| 2 | **D-6** + D-6a authoritative availability number | — | Foundation for pickup + lifecycle + overbooking rules |
| 3 | **D-8** concurrency guarantee | — | Foundation for pickup atomicity (D-9) |
| 4 | **D-5** event consumption | — | Foundation for D-4 and availability invalidation |
| 5 | **D-2** schema reproducibility | user action: live DB verification | Gates every schema-shaped decision (D-15, D-3, D-13) |
| 6 | **D-15** block lifecycle | D-2, D-6 | Defines when blocks hold inventory at all |
| 7 | **D-7** pickup ↔ availability/assertion | D-6 | Defines what pickup must validate |
| 8 | **D-9** pickup transaction boundary | D-8 | Atomicity rule needs the invariant first |
| 9 | **D-10** fabricated reservation id | — | Feeds S-1 |
| 10 | **S-4** stop-sale enforcement scope | D-6 | Feeds S-1 (uniform pickup guards) |
| 11 | **S-1** allotment pickup mechanism unification | D-10, S-4, D-9 | Needs real-id + guard rules first |
| 12 | **S-5** used-voucher cancel quota policy | S-1 | Needs canonical pickup semantics |
| 13 | **D-3** cut-off wash / rolling release | D-2 | Stored fields are schema-shaped |
| 14 | **D-4** FO checkout ↔ pickup status (L-13) | D-5, D-2 | Event-driven derivation needs events + tables |
| 15 | **D-11** guest resolution | — | Local |
| 16 | **D-12** HTTP verb contract | — | Local |
| 17 | **D-13** shoulder-day schema + semantics | D-2, D-9 | Schema artifact + pickup interaction |
| 18 | **D-14** attrition | — | Local |
| 19 | **S-2** allotment overbooking | D-6 | Availability-authority consequence |
| 20 | **S-3** contract-type vocabulary | D-1 | Vocabulary of the authoritative world |
| 21 | **S-6** tenant-scoped write statements (F-8) | — | Cross-cutting invariant |

---

## 2. Decisions

### D-1 — Which GBA data world is authoritative?

**Current state:** Two complete worlds exist — legacy (`allotment`, `block_*`) and new (`allotment_contracts`, `group_blocks`, `group_block_daily_allocations`, `group_pickups`, `allotment_vouchers`, `allotment_daily_quotas`, `allotment_stop_sales`). A type-level barrier prevents joining them: `reservations.allotment_id` is `Int` → legacy `allotment`, while `allotment_contracts.id` is `String` (`03_DATA_MODEL_AUDIT.md` D-1). Legacy tables have **no TypeScript readers or writers** — schema-only (`07_LEGACY_DEPENDENCY_MAP.md` §1). Every runtime path (adapter, repositories, handlers) reads/writes only the new world.

**Why this decision exists:** migration strategy, legacy removal (Phase 11), and "which number is real" all depend on knowing the authoritative world.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. New world authoritative | `allotment_contracts` + `group_blocks` … are the truth; legacy = Phase-11 removal candidates | Matches where all runtime code points |
| B. Legacy retained as reporting inputs | Both worlds live, legacy documented read-only | Requires maintaining two truths with no code linking them |
| C. Merge worlds | Reconcile `Int ↔ String` keys | Large mapping project; no evidence any code needs it |

**Impact:** Migration/Cutover risk (A minimizes it) · Reporting/Analytics (B would preserve legacy datamart — but nothing reads legacy today, so nothing regresses under A) · Dependency ordering (unblocks S-3, D-15, Phase 11 scope).

**Dependencies:** none (D-2 determines *mechanism*, not *which world*).

**Evidence:** `03_DATA_MODEL_AUDIT.md` D-1 · `07_LEGACY_DEPENDENCY_MAP.md` §1 (legacy has no TS readers/writers) · F-15 (`09_…` §2) · adapter + repositories write only new-world tables (`prisma-reservation-association.adapter.ts:100-120,264-287`; `group-block.repository.ts:103-202`).

**Decision status:** `DECIDED`

**Selected target rule:**
> The new GBA world (`group_bookings` / `group_blocks` / `group_block_daily_allocations` / `group_pickups` / `allotment_contracts` / `allotment_daily_quotas` / `allotment_vouchers` / `allotment_stop_sales` / `allotment_pickups`) is the sole authoritative representation of group-block and allotment data. Legacy GBA tables (`allotment`, `block_*`) are **RETAIN — read-only for historical reporting until Phase 11**; no new code may write them; their removal is a Phase 11 decision. This is an evidence-forced classification (code already behaves this way), not a business preference.

---

### D-6 — Which "available" number is authoritative: A1 (Availability) or A3 (Activities matrix)? (+ D-6a sub-decision)

**Current state:** Three different availability computations exist:
- **A1** `availability-source.adapter.ts:49-50` (WHERE: DEDUCT policy + booking `CONFIRMED/ACTIVE` + `deleted_at IS NULL`) and `:66` (block eligibility whitelist `DEFINITE`/`OPEN_FOR_PICKUP`) plus HARD_COMMITMENT + validity-window filters — feeds the snapshot service, which flags fact sources (`availability-snapshot.service.ts:83`).
- **A3** `availability-sales.controller.ts:306-335,459` — `gb.status NOT IN ('CANCELLED','CLOSED')` (not a whitelist), no `deleted_at`, no booking-status filter, no contract-type/validity filter, `ac.status IN ('ACTIVE','CONFIRMED')` where `'CONFIRMED'` is not a valid `AllotmentStatus` value; `available = max(0, physical − ooo − reserved − groupCommit − allotCommit)`.
- **F-18** `group-booking.controller.ts:95-116` — `GET /group-bookings/available-rooms` returns rooms with `room_status='AVAILABLE'` ignoring stay dates — a third, undocumented number.
The two main computations differ on **every** eligibility filter (`04_…` §3 divergence table).

**Why this decision exists:** user-facing trust, Phase 6 integration scope, and every downstream rule (D-7, D-15, S-2) need one authority.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Availability (A1) authoritative; Activities matrix reimplemented on top of it | Single rule source | Matches spec T-3 ("CRS single source of truth; GBA creates no second inventory engine", `05_…:109`) |
| B. Keep both, relabel as different views ("sellable" vs "physical − commitments") | Honest labels, two implementations forever | Divergent answers persist; every future change must touch both |
| C. Align A3's filters to A1 with a minimal SQL patch | Smallest change | Still two implementations; drift will recur |

**Impact:** Inventory/Availability (single vs double truth) · Spec conformance (T-3) · UX/Operations (two screens can disagree) · Phase 6 dependency ordering (consumers integrate against one API) · Implementation complexity (A is largest, B smallest).

**Dependencies:** none. Feeds: D-7, D-15, S-2, F-18 closure.

**Evidence:** `availability-source.adapter.ts:49-50,66,97-102` · `availability-sales.controller.ts:306-335,459` · `04_AVAILABILITY_INTEGRATION_MATRIX.md` §3 (row-by-row divergence) · F-4, F-18 (`09_…` §2) · spec T-3 `05_…:109`.

**Decision status:** `DECIDED`

**Selected target rule:**
> **Availability (the A1 computation, behind the snapshot/assertion services) is the sole authority for sellable room-type capacity.** No other component may compute or publish an independent "available" number. The Activities availability matrix (A3) and the `available-rooms` endpoint (F-18) must derive from, or be relabeled as views over, the Availability computation — they may not carry their own eligibility filters. `Inventory quantity`, `selling permission` (stop sale/restrictions), and `reservation commitment` remain separately tracked facts; only their *combination* into "sellable" is single-sourced.

**Sub-decision D-6a — transition path for A3 (mechanism):**

| Option | Assessment |
|---|---|
| A. Reimplement A3 on top of Availability | Correct end state (spec-conformant); larger change |
| B. Relabel A3 UI as a distinct view + freeze its logic | Honest immediately; keeps two numbers until Phase 6 |

**D-6a status:** `RECOMMENDED — USER CONFIRMATION REQUIRED` — **Recommend A**, with B allowed only as an explicitly-labeled interim (label must state "derived view, not sellable availability"). Scope note: whichever is chosen, F-18's date-blind endpoint is retired under the D-6 rule.

---

### D-8 — Concurrency model: which guarantee do we adopt?

**Current state:** `version` columns exist on all GBA aggregates/children but every write uses unconditional `{ increment: 1 }` and **no write path uses a version predicate or `FOR UPDATE`** (`group-block.repository.ts:134,168`; `allotment.repository.ts:153`; `06_…` §2). Concurrent pickups can lose increments — i.e., `picked` can silently under-count (C-1, F-3). Natural keys `(hotel_id, stay_date, room_type)` exist on the daily tables (`schema.prisma:17130,17215` — the foundation for any guard).

**Why this decision exists:** `picked ≤ contracted` is the module's core invariant; without a stated guarantee, pickup correctness is undefined under concurrency.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Conditional update (`WHERE version = expected`) + retry | Matches spec T-5 primary pattern | Requires threading expected version through commands |
| B. Pessimistic lock per aggregate at load | Simple mental model | Lock held across save; contention on pickup hot spots |
| C. Accept last-write-wins | None | **Violates the invariant** — lost updates under-count `picked` and can over-sell |

**Impact:** Concurrency/Integrity (this is the decision's core) · Inventory/Availability (lost increments → double-sold rooms) · Performance (lock strategy) · Spec conformance (T-5) · Testability (deterministic conflict tests required under A/B).

**Dependencies:** none. Feeds D-9 (atomicity must also be serialized), D-15, S-1.

**Evidence:** `group-block.repository.ts:134,168` · `allotment.repository.ts:153` · `06_CONCURRENCY_AND_TRANSACTION_AUDIT.md` §2 (C-1: no predicate anywhere) · spec T-5 `05_…:111` (`:1215-1227`, `:1541-1562`) · natural keys `09_…` §3.

**Decision status:** `DECIDED` (guarantee) — mechanism selection `DEFERRED` to the implementation plan (optimistic A recommended by spec T-5; pessimistic B acceptable — this is a technical mechanism, out of scope for this phase).

**Selected target rule (concurrency invariants only — no mechanism syntax):**
> 1. **No lost updates:** a write to pickup/allocation/quota counters may not overwrite a concurrent write. If two writers race, exactly one succeeds and the other is rejected or retried — never silently merged.
> 2. **Conflict is an error, not an overwrite:** a conflicting write surfaces as a CONFLICT failure to the caller (spec T-5 `CONFLICT` semantics); silent last-write-wins is prohibited.
> 3. **Core inequality:** per `hotel_id` + stay date + room type: `picked ≤ contracted − released` (block) / `picked + released ≤ quota` (allotment) must hold after **every** committed operation, including under concurrency.
> 4. **Counters change exactly once per business event** (one pickup → one increment; one cancel → one decrement); retries must be idempotent, never double-applied.
> 5. **Version monotonicity:** any version/sequence used for detection of staleness advances strictly forward; a write based on stale state must not commit.

---

### D-5 — What consumes the 22 GBA events?

**Current state:** 22 `IntegrationEvent`s are defined and published outbox → BullMQ `events` queue → `events.consumer.ts`, whose switch handles only `reservation.*` and logs `no handler` for everything else (`group-allotment.events.ts:3-396`; `events.consumer.ts:46-57`; `02_…` §4). Net effect: audit-trail churn with zero consumers (F-19, B-10). Availability is never notified (B-3, F-5: `IInventoryCommitmentPort` dead).

**Why this decision exists:** spec T-3 requires GBA to **notify** Availability/CRS on create/cancel/wash + voucher intake; D-4 (closing L-13) requires reservation events to reach GBA. Either events have consumers or the domain is architecturally broken.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Add handlers: availability cache invalidation + analytics projection | Minimal useful set | Closes F-5's availability leg |
| B. Publish externally (channels/CRS) per spec T-10 | Full spec conformance | Needs target-system contract (later phase) |
| C. Stop emitting until consumers exist | Removes churn | Contradicts T-3/T-10; loses audit trail |

**Impact:** Inventory/Availability (invalidation timing) · Spec conformance (T-3, T-10) · Dependency ordering (gates D-4) · Performance (outbox churn) · Reporting/Analytics (projection consumers).

**Dependencies:** none. Feeds: D-4, D-7 (availability refresh on pickup).

**Evidence:** `events.consumer.ts:46-57` · `group-allotment.events.ts:3-396` · `inventory-commitment.port.ts` (0 call sites) · `05_…:134` (B-10), `05_…:127` (B-3) · spec T-3 `05_…:109`, T-10 `05_…:116`.

**Decision status:** `DECIDED`

**Selected target rule:**
> GBA domain events are part of the contract (T-10), not optional chatter. **Every published GBA event must have at least one registered consumer; events with no consumer must not be published.** The mandatory consumer set includes: (a) availability cache invalidation — any block/allotment mutation that changes held or picked quantities invalidates the affected hotel/room-type/date facts; (b) the GBA-side reservation-event consumer required by D-4. External publication (CRS/channels per T-10) is **scope: implementation of a later integration step** and is not required to close this decision — but the "publish without consumer" state is prohibited.
> Mechanism (outbox → queue → handler) is unchanged; this is a rule about coverage, not plumbing.

---

### D-2 — How do the new GBA tables get into the database reproducibly?

**Current state:** Zero migrations create `group_bookings`/`group_blocks`/`group_block_daily_allocations`/`group_pickups`/`allotment_contracts`/`allotment_daily_quotas`/`allotment_vouchers`/`allotment_stop_sales`/analytics (`01_…` §3.2), while runtime code executes raw SQL against them — including `allotment_pickups` (`create-allotment-pickup.handler.ts:129-141`) and `group_blocks.shoulder_days_before/after` (`add-shoulder-days.handler.ts:113-125`), which exist in **no** migration and (for the table) in no Prisma model (`03_…` §2). The repo otherwise follows a migrate-based workflow (49 migrations). Whether these objects exist in the **live** database has not been verified (no DB access during the audit).

**Why this decision exists:** every schema-shaped decision (D-15 columns, D-3 fields, D-13 shoulder columns) and any future migration depends on a reproducible schema story. It is the top gate in `09_…` §1.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Forward migration(s) matching verified live state | Consistent with existing workflow | **Prerequisite: verify live DB first** (read-only `information_schema` check) |
| B. Accept `db push` for this domain | Fast | Diverges from the 49-migration workflow; unreproducible environments |
| C. Remove shadow artifacts from code | Honest code | Changes pickup + shoulder feature behavior — a behavior change this phase forbids, and premature before live verification |

**Impact:** Migration/Cutover risk (core) · Data honesty (schema must match executed SQL) · Dependency ordering (gates D-15, D-3, D-13, D-4) · Rollback safety (migrations are replayable; push is not) · Spec conformance (workflow).

**Dependencies:** **user/ops action prerequisite** — read-only verification: does `information_schema` contain the new GBA tables, `allotment_pickups`, and `group_blocks.shoulder_days_*` in the target environment? Feeds: D-15, D-3, D-13, D-4.

**Evidence:** `01_FORENSIC_AUDIT.md` §3.2 · `03_DATA_MODEL_AUDIT.md` §2 · `create-allotment-pickup.handler.ts:129-141` · `add-shoulder-days.handler.ts:113-125` · `remove-shoulder-days.handler.ts:31,100` · F-1 (`09_…` §2).

**Decision status:** `RECOMMENDED — USER CONFIRMATION REQUIRED`

**Domain requirement (decided regardless of mechanism):**
> Every table/column executed by runtime code must be **declared** — reproducible from version-controlled schema artifacts in a fresh environment. Code may not depend on objects that exist only in one live database.

**Recommended rule (pending confirmation):**
> **Option A — forward migration(s) reconstructing the exact live state, executed only after the read-only live-DB verification above.** If verification shows objects are absent live (code is failing silently today), that fact is reported back before choosing between A and C. Option B (accepting `db push` for GBA) is explicitly not recommended for a system with 49 migrations.

---

### D-15 — Block lifecycle: how does a block become `DEFINITE` / `OPEN_FOR_PICKUP`?

**Current state:**
- Blocks are created `DRAFT` (`group-block.aggregate.ts:402` `static create` → `GroupBlockStatus.DRAFT`; `08_…` cites `schema.prisma:17085`).
- Transition methods exist and are guarded — `confirm()` `:131`, `openForPickup()` `:138`, `close()` `:144`, `cancel()` `:151`, each via `assertTransition` `:119-123` — but **no command, controller, or test invokes them** (command inventory: 25 command directories, none named confirm/open/close-block; `confirm-group-booking` acts on bookings, not blocks).
- A complete lifecycle state machine exists twice: `lifecycle.service.ts:24-35` and `allocation-status.value-object.ts:67-74` (`DRAFT→TENTATIVE→DEFINITE→OPEN_FOR_PICKUP→CLOSED`, plus `→CANCELLED`). The three `Group*LifecycleService`s are registered in the module (`group-allotment.module.ts:69-71,118-120`) but **injected by no handler**.
- The machine is *doubly* unreachable: even if `confirm()` were called from `DRAFT`, it asserts `→ DEFINITE`, which the table only permits from `TENTATIVE` — and the aggregate has **no `tentative()` method at all** (grep: no `TENTATIVE` transition method).
- Consequence: Availability counts only `DEFINITE`/`OPEN_FOR_PICKUP` blocks (`availability-source.adapter.ts:66`) → command-created blocks contribute **0** consumption to A1. Allotments by contrast are `activate()`d at creation (`create-allotment.handler.ts:49`) and do count.
- Pickup does **not** check block status: `create-group-pickup.handler.ts:29-42` loads the block and calls `createPickup` with no status guard — pickups succeed on `DRAFT` blocks.
- `washAllocation()` (`group-block.aggregate.ts:332`) is likewise uncalled (ties D-3).

**Why this decision exists:** while unresolved, A1's block consumption is inert (F-6b) and "held inventory" is fiction for blocks.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Explicit lifecycle commands + routes (spec T-6) | Restores intended behavior | Requires permissions + tests (implementation scope); needs the missing `DRAFT→TENTATIVE` step resolved |
| B. Auto-advance on a trigger (e.g., booking confirm → `DEFINITE`) | No new UI | Implicit state change; must define which trigger — and a "confirm" is a business act someone must perform or explicitly delegate |
| C. Change A1 eligibility to include `DRAFT`/`TENTATIVE` | One-line-ish | **Rejected:** misrepresents unconfirmed commitments as held inventory — breaks D-6's single authority |
| D. Do nothing | — | **Rejected:** the DEDUCT feature stays inert; spec T-6 unimplementable |

**Impact:** Inventory/Availability (blocks start counting) · Spec conformance (T-6) · UX/Operations (someone must confirm/open/close blocks) · Security/Permissions (new transition actions need permission definitions — implementation) · Testability (state-machine tests) · Dependency ordering (depends D-2 for schema honesty, D-6 for eligibility trust).

**Dependencies:** D-2 (migration must include block tables/columns), D-6 (A1 eligibility rules are fixed by D-6). Feeds: D-7 (pickup eligibility), D-3 (`washAllocation` path).

**Evidence:** `group-block.aggregate.ts:119-156,332,402` · `lifecycle.service.ts:24-35` (transition table) · `allocation-status.value-object.ts:67-74` · `group-allotment.module.ts:69-71,118-120` (registered, never injected) · command inventory `application/commands/` (25 dirs; no block-status command) · `availability-source.adapter.ts:66` · `create-allotment.handler.ts:49` · `create-group-pickup.handler.ts:29-42` (no status guard) · F-6b (`09_…` §2) · spec T-6 `05_…:112` (`:1081-1099`), spec lifecycle `gba-domain-spec.md:69`.

**Decision status:** `DECIDED`

**Selected target rule:**
> 1. Block status changes **only through explicit, validated lifecycle operations** (spec T-6) — never implicitly and never by widening availability eligibility. Target chain: `DRAFT → TENTATIVE → DEFINITE → OPEN_FOR_PICKUP → CLOSED`, with `→ CANCELLED` permitted from any pre-closed state; the missing `DRAFT→TENTATIVE` step must exist as a first-class transition (the current table allows it; the aggregate lacks the method).
> 2. **Only `DEFINITE` and `OPEN_FOR_PICKUP` blocks hold inventory** in Availability (A1's whitelist is correct and stays).
> 3. **Pickup requires an open block:** a pickup may only be created against a block in `OPEN_FOR_PICKUP` (or, at minimum per implementation choice, `DEFINITE`); `DRAFT`/`TENTATIVE`/`CLOSED`/`CANCELLED` blocks reject pickups.
> 4. Options C and D are rejected. The concrete command set, routes, and permissions are implementation scope — not decided here.

---

### D-7 — Should GBA pickup validate against transient availability?

**Current state:** Block pickup checks only intra-block capacity (`group-block.aggregate.ts:234-242` allocation existence + `canPickup`); allotment pickup checks only intra-quota (`allotment.aggregate.ts:419-431`). Neither calls any Availability/assertion service. `IInventoryCommitmentPort` is defined but unwired (0 call sites — `05_…:127` B-3). Pickup-created reservations are inserted raw with status `CONFIRMED` (`prisma-reservation-association.adapter.ts:100-120,264-287`), bypassing the reservation assertion pipeline entirely (matrix A5; compounds L-14).

**Why this decision exists:** the Phase 2 assertion engine exists precisely to stop reservations from bypassing balances; pickup is the largest remaining bypass.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Wire pickup to the assertion engine | Every pickup-created reservation passes through normal reservation availability rules | Closes the A5/L-14 bypass; aligns with spec T-3 ("query CRS availability during pickup") |
| B. Exempt block pickup (consumes block pool only) | Matches current code | Requires documenting that pickup **ignores** transient balances — two disjoint inventory truths again (conflicts with D-6) |
| C. Wire only `DEDUCT_INVENTORY` blocks | Middle path | Non-DEDUCT still bypasses; keeps two code paths |

**Impact:** Inventory/Availability (double-count elimination) · Reservation lifecycle (pickup reservations become first-class) · Spec conformance (T-3 query duty, T-4 containment invariants) · Implementation complexity (port wiring) · Dependency ordering (needs D-6 authority; D-15 for eligible block states).

**Dependencies:** D-6 (which authority pickup queries), D-15 (eligible block status). Feeds: D-9 (the assertion runs inside the atomic unit).

**Evidence:** `group-block.aggregate.ts:234-242` · `allotment.aggregate.ts:419-431` · `inventory-commitment.port.ts` (dead) · `prisma-reservation-association.adapter.ts:100-120,264-287` (raw `CONFIRMED` insert) · `04_…` §1 (A5), `05_…:127` (B-3) · spec T-3 `05_…:109`, T-4 `05_…:110` (`:1128-1136`).

**Decision status:** `DECIDED`

**Selected target rule:**
> **Pickup validation is a two-layer check, never a substitute for the other layer:** (1) the contract layer — the allocation/quota must have capacity for the requested dates/categories (intra-pool); (2) the reservation layer — the reservation created by pickup must pass the **same** availability/assertion lifecycle as any other reservation (never bypassed, regardless of policy). Block-held capacity and transient sellable capacity are different facts (D-6 vocabulary), but pickup must consult both. Options B and C are rejected as re-creating dual inventory truths.
> Open mechanism detail (assert-at-write vs read-time derivation) is deliberately **not** chosen here — it belongs to the implementation plan; the *rule* is that no pickup path may skip either layer.

---

### D-9 — Pickup transaction boundary

**Current state:** Group pickup commits in separate units: reservation insert (adapter, no transaction — `create-group-pickup.handler.ts:44`) → folio charge (`:65`) → `repo.save(block)` (`:80`). If step 3 fails, a reservation exists with **no counter increment** → Availability double-count window (F-2, C-2). Allotment pickup: in-memory quota deduct + save (`create-allotment-pickup.handler.ts:65-81`) happens **before** reservation creation (`:92-112`); on reservation failure it performs a **manual compensating rollback** (`:114-123`, `rollbackQuota` `:165-185`) that itself swallows errors (`:183`), then inserts the pickup row afterwards (`:129-141`) — a fourth non-atomic step. The adapter performs 1 reservation + 2 folios + N routing inserts with no transaction (`prisma-reservation-association.adapter.ts:100-152,264-319`).

**Why this decision exists:** defines the correctness contract for every pickup change; without it, D-8's invariant can be violated by *ordering* rather than concurrency.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. One transaction spanning reservation + counters + pickup row | All-or-nothing | Requires the adapter to accept a transaction handle (currently uses `this.prisma` directly) — implementation detail |
| B. Outbox/saga with compensating cancel | Event-driven style | Heavier; correct only if compensation is itself reliable (today's `rollbackQuota` is not — it logs and swallows, `:183`) |
| C. Reorder (counters first) + compensation | Cheaper | Still relies on compensation; never a true guarantee |

**Impact:** Concurrency/Integrity (core) · Inventory/Availability (eliminates double-count window) · Billing/Folio (folio post must not orphan) · Rollback safety (A is atomic; B/C depend on compensations that have proven unreliable) · Implementation complexity (adapter transaction plumbing).

**Dependencies:** D-8 (serialization + invariant must hold inside the boundary). Feeds: D-13 (shoulder writes join the same discipline), S-1.

**Evidence:** `create-group-pickup.handler.ts:44,65,80` · `create-allotment-pickup.handler.ts:65-81,114-123,129-141,165-185` · `prisma-reservation-association.adapter.ts:100-152,264-319` · `06_CONCURRENCY_AND_TRANSACTION_AUDIT.md` C-2/§3.1 · F-2 (`09_…` §2).

**Decision status:** `DECIDED`

**Selected target rule:**
> **A pickup is one atomic business operation.** The reservation row, the pickup/association record, and the counter changes (block `picked_qty` / allotment `quota.picked`) commit together or not at all. Partial commit is prohibited: there must never be a reservation without its counter increment (nor the reverse), in any failure mode. Compensating-transaction scripts (current `rollbackQuota`) are not a substitute for atomicity; if an implementation chooses saga-style compensation instead (option B), the compensation must be reliable and observable — but the default target is option A semantics: **all-or-nothing**. The exact mechanism (shared transaction handle vs saga) is implementation scope.

---

### D-10 — Fabricated voucher reservation id

**Current state:** `ConsumeAllotmentVoucherHandler` does `const reservationId = command.data.reservationId || \`RES-${Date.now()}\`` (`consume-allotment-voucher.handler.ts:26`) — a fabricated, non-existent id is stored as if real, then `allotment.consumeVoucher` marks the voucher `USED` (`:27`). No reservation is verified or created (B-11: "actively violated"). The frontend posts to this endpoint with a possibly-empty body (`group-allotment.api.ts:303`).

**Why this decision exists:** voucher↔reservation reporting is untrustworthy while fabrication is possible; S-1's canonical flow depends on the answer.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Require `reservationId`; reject if missing | Honest linkage | May break existing UI callers until they adapt |
| B. Create the reservation as part of consume (like pickup does) | Matches spec T-3 voucher intake (notify on intake) + T-9 association | Larger handler; subsumes A's honesty |
| C. Allow null / store null | Honest about state | Loses linkage entirely — voucher consumed with no reservation is a business hole |

**Impact:** Reporting/Analytics (voucher↔reservation integrity) · Reservation lifecycle (B makes consume an intake operation) · UX/Operations (callers must pass or receive a reservation) · Spec conformance (T-3, T-9) · Dependency ordering (feeds S-1).

**Dependencies:** none. Feeds: S-1 (canonical mechanism), D-9 (if consume creates a reservation, atomicity rule applies).

**Evidence:** `consume-allotment-voucher.handler.ts:26-27` · `05_…:135` (B-11) · `group-allotment.api.ts:303` · spec T-9 `05_…:115`, T-3 `05_…:109`.

**Decision status:** rule `DECIDED` — mechanism `RECOMMENDED — USER CONFIRMATION REQUIRED`

**Decided rule:**
> **Fabricated reservation identifiers are prohibited.** A voucher's `reservationId` is either a real reservation id that exists in this hotel's data, or the field is null/absent — never a synthetic token (`RES-…`). Voucher consumption asserts a reservation linkage that is true.

**Recommended mechanism:**
> **Option B — voucher consumption creates (or links an already-created real) reservation as part of consume**, mirroring the pickup flow; reject the operation if the reservation cannot be created. Option A (require caller-supplied id) is acceptable as an interim if UI callers already supply one — **confirm which caller pattern exists before locking this in** (verification is read-only frontend inspection).

---

### S-4 — Stop-sale enforcement must be uniform across every allotment intake path (supplementary — from F-9/§8F)

**Current state:** Stop sale is enforced on **exactly one** path: voucher issue (`allotment.aggregate.ts:245-247` throws when `isStopSaleActive`, inside the issue loop `:242-252`). The direct pickup path `pickupQuota` (`:419-431`) checks ACTIVE + quota only — **no stop-sale check**. Guard formulas also diverge: issue uses `quota − picked` (`:248`), pickup uses `quota − picked − released` (`:423`). Spec: stop sale is a *selling permission* restriction (glossary `gba-domain-spec.md:38`, `:437` "Remaining = quota − picked, **0 if stop sale**"), applies to allotment contracts only (T-8 `05_…:114`), and does **not** change quantities or release rooms (`:628-646`).

**Why this decision exists:** two intake paths with different guard sets means stop sale is trivially bypassable — a yield-protection rule that fails its purpose.

**Options:**

| Option | Assessment |
|---|---|
| A. Stop sale blocks **all** allotment intake (issue + direct pickup), allotment-only scope, quantity untouched | Spec-conformant; single rule |
| B. Keep issue-only enforcement (current) | Bypassable; undocumented asymmetry |
| C. Stop sale also blocks group-block pickups | Violates T-8 (blocks explicitly out of stop-sale scope) |

**Impact:** Inventory/Availability (selling permission honored) · Spec conformance (T-8, `:628-646`) · UX/Operations (revenue manager's lift action must work everywhere) · Testability (one guard, one test matrix).

**Dependencies:** D-6 (permission vs quantity vocabulary). Feeds: S-1 (uniform guards).

**Evidence:** `allotment.aggregate.ts:245-247` (issue enforces) vs `:419-431` (pickup does not) · `:248` vs `:423` (divergent formulas) · spec T-8 `05_…:114` (`:628-632`), glossary `:38`, `:437`, `:628-646`.

**Decision status:** `DECIDED`

**Selected target rule:**
> **Stop sale is a selling-permission fact for allotment contracts only (T-8). While a stop sale is active for a date + category, no allotment intake may occur on ANY path — voucher issue and direct pickup alike. Stop sale never changes quantity and never releases rooms (inventoried quantity and commitment counters are untouched); lifting the stop sale restores selling, not inventory. Group-block pickups are never stop-sale restricted.** All intake guards use one formula: remaining = quota − picked − released (the stricter of the two current formulas), evaluated after the stop-sale check.

---

### S-1 — Which allotment pickup mechanism is canonical? (supplementary — two live mechanisms)

**Current state:** Two independent consumption mechanisms coexist:
1. **Voucher flow:** `create-allotment-voucher` → `issueVoucher` increments `quota.picked` at **issue** (`allotment.aggregate.ts:242-257`, stop-sale enforced `:245-247`) → `consume-allotment-voucher` marks `USED` and fabricates/links a reservation id (`consume-allotment-voucher.handler.ts:26-27`) — consume does **not** create a reservation and does **not** touch quota.
2. **Direct pickup flow:** `create-allotment-pickup` → `pickupQuota` increments `quota.picked` at **pickup** (`allotment.aggregate.ts:419-431`, no stop-sale check) → creates reservation raw (`create-allotment-pickup.handler.ts:92-112`) → inserts `allotment_pickups` row (`:129-141`).
Both increment `picked` exactly once per flow (issue-time vs pickup-time), but: two record models (`allotment_vouchers` vs `allotment_pickups`), two guard formulas (S-4), two stop-sale behaviors (S-4), and B-9 — association lives partly in voucher state and partly in a pickup table (`05_…:133`). The spec's association model is pickup records (T-9 `05_…:115`, spec `:871-900`).

**Why this decision exists:** reporting, accounting, and D-10 cannot be made coherent while "a pickup" means two different things.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. One canonical pickup record; voucher = authorization artifact | Quota held at issue (stop-sale checked); consume MUST yield a pickup record linked to a real reservation; direct pickup yields the same record without a voucher; guards uniform | Spec-conformant (T-9); unifies reporting; subsumes D-10-B |
| B. Voucher flow is the only canonical path; deprecate direct pickup | Simplest rule | Removes legitimate manual/FRONT-DESK intake — capability loss; needs business confirmation |
| C. Keep both with mutual-exclusion guard | Least change | Permanently two models; every rule written twice |

**Impact:** Reservation lifecycle · Reporting/Analytics (one pickup ledger) · Spec conformance (T-9) · Concurrency/Integrity (one counter path) · Implementation complexity · Dependency ordering (consumes D-10, S-4, D-9).

**Dependencies:** D-10 (real ids), S-4 (uniform guards), D-9 (atomicity applies to whichever flow creates records). Feeds: S-5 (cancel policy), D-4 (pickup status derivation targets the canonical record).

**Evidence:** `allotment.aggregate.ts:242-257,419-431` (two increment points) · `consume-allotment-voucher.handler.ts:26-27` · `create-allotment-pickup.handler.ts:92-141` · `05_…:133` (B-9 both patterns live) · spec T-9 `05_…:115`.

**Decision status:** `RECOMMENDED — USER CONFIRMATION REQUIRED`

**Recommended rule (pending confirmation):**
> **Option A.** One canonical entity — the **pickup record** — represents "rooms consumed against an allotment," always linked to a real reservation (D-10), guarded uniformly (S-4), and mutated atomically (D-9). Vouchers remain the *authorization* artifact of the contractual (CRS) flow: quota is held at issue, and **consumption without a resulting pickup record is impossible**. Direct pickup (no voucher) remains supported for manual intake but writes the same record under the same guards. Confirm option A (or choose B if direct/manual intake is not a real business need).

---

### S-5 — Cancelling a USED voucher: restore quota or not? (supplementary — from B-12)

**Current state:** `allotment.cancelVoucher` (`allotment.aggregate.ts:294-315`): if the voucher is `USED`, it does **nothing to quota** — comment at `:301-302` admits policy is undecided ("For now, mark as cancelled without restoring quota for used vouchers"); if `ISSUED`, it restores `quota.picked` (`:304-312`). The spec is silent on used-voucher cancellation (B-12, `05_…:136`).

**Why this decision exists:** cancelling a used voucher usually means the underlying reservation was cancelled — and whether quota returns depends on whether the reservation (not the voucher) is the source of truth for consumption.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Quota state follows the reservation (derive) | Used-voucher cancel: quota restores **iff** the linked reservation is cancelled; otherwise quota stays consumed | Consistent with D-4 derivation + D-10 real ids; no double-source of truth |
| B. Never restore for used vouchers (current) | Simple | Cancelling the only reservation of a pickup permanently burns quota — likely wrong operationally |
| C. Always restore on used-voucher cancel | Symmetric with issued | Can restore quota while a live reservation still exists → over-sells the allotment |

**Impact:** Inventory/Availability (quota correctness) · Reservation lifecycle (voucher vs reservation authority) · Reporting · Dependency ordering (needs S-1, D-10, D-4).

**Dependencies:** S-1 (canonical pickup record), D-10 (real linkage required to even evaluate A), D-4 (reservation-lifecycle derivation machinery).

**Evidence:** `allotment.aggregate.ts:298-312` · `cancel-allotment-voucher.handler.ts:26-31` · `05_…:136` (B-12, spec silent).

**Decision status:** `USER DECISION REQUIRED`

**Recommendation:** **Option A** — quota consumption follows the reservation lifecycle (cancel reservation ⇒ restore quota; voucher state alone never decides). Options B and C are both wrong in one direction; only A keeps one source of truth. Note: B and C are workable *temporary* stances if the user explicitly accepts their failure modes.

---

### D-3 — Cut-off wash and rolling release: build now, later, or never?

**Current state:** `cutoff_date` is stored and never read by any rule; `release_days_before` / `is_rolling_release` are stored and never evaluated; `ReleaseWindow.isWithinReleaseWindow` has 0 call sites (`release-window.value-object.ts:26`); no scheduler exists for wash. Manual release exists and works (`release-block-allocation`, `release-allotment-allocation` commands; aggregate guards `group-block.aggregate.ts:310-330`, `allotment.aggregate.ts:401-415`). The aggregate's `washAllocation` (`group-block.aggregate.ts:332`) — the event-emitting wash path — has no callers (F-6b). Consequence: **a block/allotment holds inventory indefinitely until manually released** (B-1/B-2). Users can enter cutoff dates that do nothing (data honesty problem). Spec: cut-off wash returns unpicked rooms at cutoff (T-1), rolling release washes N days pre-arrival (T-2), GUARANTEED cannot wash (`gba-domain-spec.md:1146`).

**Why this decision exists:** stored-but-unenforced fields are an operational trap; the spec makes wash a core guarantee.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Implement scheduled wash (Temporal job — platform pattern exists) | Restores T-1/T-2 | Requires wash semantics per policy/contract type (defined below as part of the rule); touches D-15 (`washAllocation` path) |
| B. Manual release only; delete dead `ReleaseWindow` + release fields | Simplest, honest | Diverges from spec T-1/T-2; gives up automatic inventory reclamation |
| C. Defer to a later phase | No work now | Keeps the data trap; must be an explicit, labeled deferral |

**Impact:** Inventory/Availability (held → sellable over time — the core value) · Spec conformance (T-1, T-2) · Data honesty (fields become meaningful or are removed) · UX/Operations (who executes wash, and when) · Dependency ordering (D-2 schema, D-15 lifecycle, D-5 wash events).

**Dependencies:** D-2 (release/cutoff columns must be declared), D-15 (`washAllocation` lifecycle), D-5 (`CutOffWashExecuted` / `AllotmentWashExecuted` events per T-10). Feeds: D-4, availability refresh on wash.

**Evidence:** `release-window.value-object.ts:26` (0 call sites) · `05_…:125-126` (B-1, B-2) · `04_…` §4 · `group-block.aggregate.ts:332` (uncalled wash) · manual release commands (command inventory) · spec T-1/T-2 `05_…:107-108`, guaranteed-no-wash invariant `gba-domain-spec.md:1146`, `:160-166`, `:596-603`.

**Decision status:** `USER DECISION REQUIRED`

**Sub-question (date semantics):** cutoff/release columns are DATE-typed — recommend **date-granularity** wash (runs at start of the day following/preceding per rule), not an arbitrary clock time.

**Recommendation:** **Option A** — implement scheduled wash per T-1 (single wash event per block at cut-off) and T-2 (rolling N-days-before-arrival for allotments), with **GUARANTEED_BLOCK excluded from wash** (`:1146`) and wash emitting the T-10 wash events through the D-5 consumers. If the user prefers option C, it must be recorded as an explicit deferral with the data-trap acknowledged; option B is acceptable only if the user also decides the cutoff/release columns become **non-authoritative display metadata** (re-labeled in UI) rather than implied behavior.

---

### D-4 — FO checkout ↔ pickup status: keep, move, or replace? (Phase 3 L-13)

**Current state:** Front Office checkout writes pickup status directly: `allotment_pickups`/`group_pickups` → `CHECKED_OUT` via raw SQL inside `try/catch` (`check-out.handler.ts:390-392,479-481`); failures are logged and swallowed → status can silently stay `ACTIVE` (`05_…:96-99`). No counter changes at checkout (correct — a consumed room stays consumed). The tables may not even exist in some environments (D-2). FO checkout is otherwise the **only** FO command touching GBA state. Phase 3 verdict: "KEEP until Phase 4".

**Why this decision exists:** L-13 must be closed with one owner for pickup "consumed" state; divergent writers (FO handler vs GBA) are the source of the bug class.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Keep as-is | Zero effort | Silent-failure behavior persists; FO keeps writing another domain's tables |
| B. Move into GBA via event consumer (`reservation.checked_out` → GBA handler) | GBA owns pickup state; FO stops writing GBA tables | Requires D-5 consumers |
| C. Derive pickup status from reservation status entirely (drop write semantics) | Single source of truth: pickup status is a **projection** of its reservation | Largest change; removes divergence permanently |

**Impact:** Front-office ops (FO stops writing GBA) · Reporting/Analytics ("consumed" consistency) · Dependency ordering (needs D-5, D-2) · Rollback safety · Spec conformance (event-driven, T-10 style).

**Dependencies:** D-5 (reservation events must be consumed), D-2 (tables declared). Feeds: S-5 (derivation machinery reused).

**Evidence:** `check-out.handler.ts:390-392,479-481` · `05_…:95-99` (§1 checkout table) · F-16 (`09_…` §2) · L-13 (`07_…` §2).

**Decision status:** `RECOMMENDED — USER CONFIRMATION REQUIRED`

**Recommended rule (pending confirmation):**
> **Option B's structure with C's semantics — pickup consumed-state is derived from the reservation lifecycle, delivered by the GBA event consumer (D-5):**
> - `reservation.checked_out` → pickup record marked `CHECKED_OUT` (consumed); **counters unchanged** (the room was sold — it stays picked).
> - `reservation.cancelled` → pickup record cancelled **and quota/counter restored** (consumption undone).
> - FO `check-out.handler`'s direct GBA writes are removed (closes L-13) — FO owns check-out, GBA owns pickup state, the reservation event links them.
> - Failures are no longer silent: consumer failures retry via the queue (D-5), surfacing instead of being swallowed.
>
> Confirmation needed because this retires a Phase-3 "KEEP" verdict.

---

### D-11 — Guest resolution by `full_name ILIKE`

**Current state:** Both adapter flows match guests by hotel + case-insensitive **full name**, else create: `prisma-reservation-association.adapter.ts:83-97` (block pickup), `:245-260` (allotment pickup). Same-name guests (common: "John Smith") silently merge across reservations (F-17, B-13). The spec is silent on guest resolution.

**Why this decision exists:** guest-data integrity; merging distinct people is a correctness/privacy problem, but creating duplicates on every pickup is an operations problem.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Keep (name convenience) | Zero change | Silent merging of distinct guests |
| B. Always create new guest | No merging | Duplicate guests proliferate with every pickup |
| C. Match by explicit `guestId` or (email + hotel) first; otherwise create | Identity-based | Needs caller fields — API change (implementation) |

**Impact:** Guest data (core) · Reservation lifecycle · UX/Operations (duplicate vs merge tradeoff) · Spec conformance (silent — free choice) · Dependency ordering (independent).

**Dependencies:** none.

**Evidence:** `prisma-reservation-association.adapter.ts:83-97,245-260` · F-17 (`09_…` §2) · `05_…:137` (B-13, spec silent).

**Decision status:** `USER DECISION REQUIRED`

**Recommendation:** **Option C** — never merge on `full_name ILIKE`. Resolve by explicit guest id when supplied; else by hotel-scoped email match; else create a new guest. Options A and B each fail deterministically (silent merge vs duplicate explosion).

---

### D-12 — HTTP verb contract for block release

**Current state:** Frontend calls `api.put('/group-bookings/…/release')` (`group-allotment.api.ts:146`); controller declares `@Post('…/release')` (`group-booking.controller.ts:247`) → the block-release action returns **405** (F-13).

**Why this decision exists:** the action is broken; one side must change (this phase changes neither — it decides which side is right).

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Frontend switches to POST | Controller stays canonical | One-line frontend fix (implementation scope) |
| B. Controller accepts PUT too | Both work | Two verbs for one action — contract ambiguity |

**Impact:** UX/Operations (feature works) · Spec conformance (REST idiom: release is a command → POST) · Implementation complexity (both trivial).

**Dependencies:** none.

**Evidence:** `group-allotment.api.ts:146` vs `group-booking.controller.ts:247` · F-13 (`09_…` §2).

**Decision status:** `DECIDED`

**Selected target rule:**
> **`POST` is the canonical verb for release** (a release is a state-changing command, matching every other GBA command route). The frontend caller is corrected to `POST`; the controller is not duplicated with a `PUT` alias. (Fix itself is implementation, blocked only by phase rules, not by any open decision.)

---

### D-13 — Shoulder-day schema artifacts & semantics

**Current state:**
- **Schema:** `add-shoulder-days`/`remove-shoulder-days` execute raw SQL against `group_blocks.shoulder_days_before/after` (`add-shoulder-days.handler.ts:113-125`, `remove-shoulder-days.handler.ts:31,100`); the columns exist in **no migration** and are absent from `schema.prisma` (`03_…` §2, D-4) — part of D-2.
- **Semantics today:** shoulder days extend the block's `arrival_date`/`departure_date` range (`add-shoulder-days.handler.ts:113-125`) and insert daily allocations **with `contracted_qty = 0`** (`:96-104`, `ON CONFLICT DO NOTHING`) — placeholders that hold no inventory until quantities are set elsewhere. The handler publishes **no domain events** (eventBus injected `:17`, never called) → availability is not invalidated by shoulder changes. It is non-transactional across allocations + block update, overwrites rather than accumulates the opposite-direction shoulder count (`:110-111`), and has no max-days guard.
- **Removal:** `remove-shoulder-days` deletes allocations with **no pickup check** and doesn't recompute block rollups (F-11, `remove-shoulder-days.handler.ts:79-94`) — while normal allocation removal *does* guard (`group-block.aggregate.ts:198-200` "Cannot remove allocation with active pickups").
- **Feature surface:** `POST …/shoulder` routes + UI exist (`08_…` D-13).

**Why this decision exists:** the columns must be declared (D-2), and the semantics must be fixed: are shoulder days ordinary allocations or a special class?

**Options (schema):**

| Option | Description |
|---|---|
| A. Add columns to Prisma model + migration | Makes code honest |
| B. Remove shoulder feature | Removes capability (routes + UI) — behavior change requiring user intent |

**Options (semantics — differentiators):** (i) shoulder days are **ordinary** allocations inside the extended range — same pickup guards, same removal guard, same events, same availability treatment; (ii) shoulder days are a **special class** — e.g., excluded from attrition base (D-14), different cutoff/wash interaction.

**Impact:** Inventory/Availability (dates now in range → A1 sees them once quantities set) · Migration/Cutover risk (D-2) · Spec conformance (spec models block date ranges; shoulder as range-extension matches) · Data honesty (qty-0 placeholders + missing events are dishonest today) · Dependency ordering (D-2, D-9 atomicity for its writes).

**Dependencies:** D-2 (columns), D-9 (writes must join the atomic discipline), D-14 (attrition base question), D-3 (cutoff/wash interaction question). Feeds: none downstream.

**Evidence:** `add-shoulder-days.handler.ts:96-104,110-125` (qty-0 inserts, date extension, no events, count overwrite) · `remove-shoulder-days.handler.ts:31,79-94,100` · `group-block.aggregate.ts:198-200` (normal guard exists) · F-11 (`09_…` §2) · `03_…` §2/D-4.

**Decision status:** `USER DECISION REQUIRED` (differentiators) — base rule `DECIDED`; schema mechanism rides on D-2.

**Decided base rule:**
> Shoulder days are **ordinary daily allocations within the extended block date range**: created with declared quantities (not silent qty-0 placeholders), mutated through the same guarded aggregate paths as core-day allocations (including the "no removal with active pickups" guard), emitting the same domain events (availability invalidation included), inside the same atomic unit (D-9), scoped to the hotel. Removal without a pickup check (F-11) is prohibited. The `shoulder_days_before/after` columns must be declared via D-2's mechanism.

**User decision required (differentiators — pick one policy):**
> - **P1:** Do shoulder-day rooms count toward the **attrition base** (D-14)? 
> - **P2:** Do cut-off/wash rules (D-3) apply to shoulder allocations exactly like core days?
> - **P3:** Are pickups allowed on shoulder dates unconditionally, or restricted (e.g., first/last-day constraints)?
>
> **Recommendation:** uniform treatment — shoulder days answer YES to P1/P2 and YES to P3 (they are ordinary days), because special-casing them recreates the dual-rule divergence D-6 forbids. Choose deliberately if shoulder days are intended as buffer (non-sellable) inventory instead.

---

### D-14 — Attrition: activate or remove?

**Current state:** `AttritionCalculationService` is provided but never injected (`group-allotment.module.ts:72,121`); `attrition_threshold` is stored (default 80) and never evaluated; no computation path anywhere (B-7, F-14). Spec T-7 defines threshold % (default 80), shortfall calculation, penalty posted to master folio (`05_…:113`).

**Why this decision exists:** a decorative column + dead service is either an unimplemented guarantee (revenue protection) or honest-schema debt.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Implement attrition report/penalty flow (T-7) | Feature work (implementation scope) | Revenue-manager guarantee restored |
| B. Remove service + threshold column | Honest schema | Loses contractual protection capability |

**Impact:** Reporting/Analytics (shortfall visibility) · Billing/Folio (penalty posting) · Spec conformance (T-7) · Data honesty · Dependency ordering (interacts D-13 shoulder base, D-3 wash timing — attrition assessed at wash per spec `:591-592`).

**Dependencies:** D-3 (attrition is assessed at wash time — spec `:591-592`), D-13 (base definition). Feeds: none.

**Evidence:** `group-allotment.module.ts:72,121` · `05_…:131` (B-7) · F-14 (`09_…` §2) · spec T-7 `05_…:113` (`:564-566`, `:1522-1530`), wash-time assessment `:591-592`.

**Decision status:** `USER DECISION REQUIRED`

**Recommendation:** **Option A — retain and implement attrition per T-7** (threshold default 80%, shortfall vs contracted at wash, penalty on master folio), because the spec treats it as a core group-biz guarantee and option B removes a revenue-protection contract term. Option B is defensible only if the business does not sell attrition-bearing contracts.

---

### S-2 — Allotment overbooking (spec open question) (supplementary)

**Current state:** The spec explicitly leaves it open: "**Should allotment contracts support overbooking (quota > physical rooms)?** … Should revenue management be allowed to over-commit allotments?" (`gba-domain-spec.md:1720`). Current code cannot overbook: `pickupQuota` hard-fails below remaining (`allotment.aggregate.ts:424-426`), and issue-path checks `quota − picked` (`:248-251`). A1 subtracts allotment commitment from physical (`availability-source.adapter.ts:49-50`), so quota > physical would drive computed availability negative-clamped — behavior undefined beyond the guard.

**Why this decision exists:** "may quota exceed physical inventory?" is pure revenue policy; code happens to answer "no" today.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. No overbooking (hard quota ≤ physical) | Current behavior, made explicit | Safe default; matches reference behavior the spec notes |
| B. Allow overbooking with policy (tolerance %, flag) | Revenue upside | Needs tolerance definition + availability override semantics — interacts D-6 authority |
| C. Defer | Unspecified | Same as today but undocumented |

**Impact:** Inventory/Availability (sell past physical or not) · Spec conformance (open question — free choice) · Reporting · UX/Operations.

**Dependencies:** D-6 (an override would need an explicit, labeled fact — not a second number). Feeds: none.

**Evidence:** `gba-domain-spec.md:1720` · `allotment.aggregate.ts:248-251,424-426` · `availability-source.adapter.ts:49-50`.

**Decision status:** `USER DECISION REQUIRED`

**Recommendation:** **Option A** — declare hard quota (no overbooking) as the rule until revenue management defines a tolerance policy; it matches existing behavior (no code surprise) and keeps D-6's authority clean. If B is chosen, the overcommit must surface as an explicit flagged fact in Availability, never as silent clamp behavior.

---

### S-3 — Contract-type vocabulary: spec vs code (supplementary)

**Current state:** Two vocabularies for the same concept:
- **Code:** `SOFT_QUOTA`, `HARD_COMMITMENT`, `FREE_SALE`, `GUARANTEED` (`allocation-status.value-object.ts:52-55`) + separate boolean `is_rolling_release` (`allotment.aggregate.ts:457`) — i.e., "rolling" is orthogonal to type.
- **Spec:** `ROLLING_RELEASE`, `GUARANTEED_BLOCK`, `FREE_SALE` (`gba-domain-spec.md:346,989`) — i.e., rolling *is* a type; and wash rules key off type (GUARANTEED cannot wash `:1146`).
A1 filters on `HARD_COMMITMENT` (`availability-source.adapter.ts:49-50`) — the code vocabulary directly drives availability, so vocabulary choice has behavioral consequences.

**Why this decision exists:** wash eligibility (D-3), availability filtering (D-6), and stop-sale scope (S-4) all key off contract type; two vocabularies mean every rule must be written twice or silently mistranslated.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Adopt spec vocabulary as canonical; publish a mapping from code values (`SOFT_QUOTA→…`, `is_rolling_release` folds into `ROLLING_RELEASE`, etc.) | One rulebook; spec-conformant | Mapping must be explicit (schema/data later — Phase 11/implementation) |
| B. Keep code vocabulary canonical; amend the spec | No data change | Spec diverges from the shipped system; T-rules keyed on spec names become ambiguous |
| C. Dual mapping forever | No decision | Permanent translation cost; drift guaranteed |

**Impact:** Spec conformance (core) · Inventory/Availability (A1's `HARD_COMMITMENT` filter must map) · Data honesty (one vocabulary) · Dependency ordering (D-1, D-6, D-3, S-4 all cite types) · Migration/Cutover risk (later renaming).

**Dependencies:** D-1 (authoritative world). Feeds: D-3 (wash-by-type), D-6 (filter vocabulary), S-4.

**Evidence:** `allocation-status.value-object.ts:52-55` · `allotment.aggregate.ts:457` · `gba-domain-spec.md:346,989,1146` · `availability-source.adapter.ts:49-50`.

**Decision status:** `RECOMMENDED — USER CONFIRMATION REQUIRED`

**Recommended rule (pending confirmation):**
> **Option A — the spec vocabulary (`ROLLING_RELEASE` | `GUARANTEED_BLOCK` | `FREE_SALE`, plus explicit soft/hard commitment semantics) is canonical for all rules and documentation**, with a published 1:1 mapping from current stored values (`SOFT_QUOTA`, `HARD_COMMITMENT`, `FREE_SALE`, `GUARANTEED`, `is_rolling_release`) applied at read/translation boundaries now and in schema later (Phase 11). Availability's contract-type filter and the wash-by-type rule are stated against the canonical vocabulary only.
> *(Confirm the mapping direction: spec-canonical (A) vs code-canonical (B) — A is recommended because every T-rule, including the "Guaranteed No Wash" invariant, is written in spec vocabulary.)*

---

### S-6 — Tenant-scoped write statements & header override (supplementary — from F-8)

**Current state:** Four statement sites omit `hotel_id` scoping: `cancel-allotment-pickup.handler.ts:66,89-94` (UPDATE/SELECT on `allotment_pickups`), adapter `linkReservation` (`prisma-reservation-association.adapter.ts:179-182`) and `unlinkReservation` (`:200-203`) — bare `WHERE id = $n`. Compounding: `x-property-id` header is accepted to override tenant context (`tenant.interceptor.ts:19-20`) and the dev property check is disabled (`resource-access.guard.ts:29`). By contrast, standard repository reads are correctly scoped (`group-block.repository.ts:23,30,36`; `allotment.repository.ts:24,30,36`). Multi-tenant isolation is a platform invariant (AGENTS.md: row-level scoping by `hotelId`).

**Why this decision exists:** one unscoped UPDATE can mutate another hotel's data — this is not a "style" issue but the tenant boundary itself.

**Options:**

| Option | Description | Assessment |
|---|---|---|
| A. Rule: every GBA read/write statement is hotel-scoped; override restricted to privileged role; dev bypass never in prod | Closes the class | Enforcement is implementation; the *rule* is decidable now |
| B. Fix only the 4 sites | Smallest | New raw-SQL sites will regress without a rule |
| C. Move scoping entirely to DB-level RLS | Strongest | Platform-wide mechanism decision — out of scope here |

**Impact:** Multi-tenant isolation (core) · Security/Permissions · Concurrency/Integrity (scoped predicate participates in conflict detection) · Implementation complexity.

**Dependencies:** none. Applies to every rule in `11_TARGET_BUSINESS_RULES.md` (hotel isolation cross-cut).

**Evidence:** `cancel-allotment-pickup.handler.ts:66,89-94` · `prisma-reservation-association.adapter.ts:179-182,200-203` · `tenant.interceptor.ts:19-20` · `resource-access.guard.ts:29` · F-8 (`09_…` §2) · positive scoping `group-block.repository.ts:23,30,36`.

**Decision status:** `DECIDED`

**Selected target rule:**
> **Every GBA read and write statement — including raw SQL — is scoped by `hotel_id` as part of its predicate, not merely filtered after load.** No mutation may target a row by bare id. The `x-property-id` header override is permitted only for an explicitly privileged role (and never silently in production); the development bypass in resource access must never be active where real tenant data exists. This rule is cross-cutting: every target rule in this phase inherits it.

---

## 3. Decisions Explicitly NOT Taken Here (routed elsewhere)

| Item | Why not a decision here | Routed to |
|---|---|---|
| F-10 — 0 tests for 25 commands | Test *implementation* is forbidden this phase; plan belongs to implementation phase | Implementation phase (pre-work) |
| F-12 — `nextBlockCode` UUID-vs-code bug (`group-block.repository.ts:212-216`) | Defect with an obvious correct behavior — no business choice | Implementation (code fix) |
| D-8 mechanism (optimistic vs pessimistic) | Technical mechanism; brief forbids locking syntax | Implementation plan (`DEFERRED`) |
| D-7 assertion mechanism (assert-at-write vs read-time) | Technical mechanism | Implementation plan |
| D-15 command/route/permission surface | Implementation detail of the decided rule | Implementation plan |
| B-9 column-vs-association-entity (spec `:1734`) | Schema removal of `reservations.group_block_id` etc. = Phase 11 migration decision; rule "pickup records are authoritative" is stated in `11_…` §9 | Phase 11 / implementation |
| Spec open questions `:1716-1728` (master-folio ownership, BEO context, analytics real-time, aggregate size limits, grid storage) | Outside GBA decision set for this phase; none block the 15 decisions | Backlog (recorded, not skipped) |
| F-1 — new tables without migration | Not a choice: verification action + D-2 decide the response | D-2 prerequisite (user action) |

---

## 4. Consistency Checks (brief §8/§9)

| Check | Result |
|---|---|
| D-1…D-15 all addressed, none skipped | **PASS** — all 15 resolved above + 6 supplementary (S-1…S-6) + 1 sub-decision (D-6a) |
| Dependencies identified and resolved in order | **PASS** — §1 graph; no decision depends on an unresolved one |
| No silent business-rule selection | **PASS** — 6 `USER DECISION REQUIRED` items carry explicit recommendations and are listed in `12_DECISION_STATUS.md` §2; no choice made without labeling |
| Block lifecycle complete & deterministic | **PASS** — D-15 (states, transitions incl. missing `DRAFT→TENTATIVE`, eligible-for-hold states, pickup eligibility) |
| Pickup/release/stop-sale/shoulder/availability semantics deterministic | **PASS** — D-7/D-9/D-10 (pickup), D-3 + guard rule (release), S-4 (stop sale), D-13 (shoulder), D-6 (availability) |
| Concurrency invariants explicit (business level only) | **PASS** — D-8: five invariants; no lock syntax; mechanism `DEFERRED` |
| Hotel isolation guaranteed for every selected rule | **PASS** — S-6 cross-cutting rule; every target rule in `11_…` inherits it |
| Data-model decisions = identity/lifecycle/ownership/relationships/invariants + fact classification only | **PASS** — no Prisma model, column, index, or migration specified anywhere |
| Phase separation: 0 code/schema/migration/backfill/API changes | **PASS** — read-only source inspection only; three docs written |
| Legacy classification labels used correctly | **PASS** — D-1: legacy = `RETAIN` (read-only) → `REMOVE AFTER CUTOVER` (Phase 11); no legacy deletion decided |

---

**End of `10_DECISION_RESOLUTION.md`.** Statuses summarized in `12_DECISION_STATUS.md`; consolidated target rules in `11_TARGET_BUSINESS_RULES.md`.

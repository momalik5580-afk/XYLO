# XYLO HOUSEKEEPING — PHASE 2 DECISION ANALYSIS RECORDS

Document: 03_DECISION_ANALYSIS_D01_D08.md
Lifecycle step: Phase 2 — Business Rules Decision
Status: COMPLETE
Date: 2026-10-20
Evidence basis: Phase 1 audit documents (`docs/audit/Housekeeping/02–05`) + targeted source re-verification of load-bearing facts (`schema.prisma`, `housekeeping.service.ts`, FO discrepancy stack, inventory supply-request module). Evidence distinction preserved: nothing is runtime-proven.

Conventions: decision IDs `HKD-###`; fact labels **F** (verified fact), **I** (inference), **U** (unknown/open). Statuses: PROPOSED / PENDING_OWNER_DECISION / APPROVED / REJECTED / DEFERRED / BLOCKED_BY_EVIDENCE. Owner-approved records cite the 2026-10-20 interactive session (recorded in 01 §6).

- [§1 Cluster D-01 — Room lifecycle and re-entry policy](#1-cluster-d-01--room-lifecycle-and-re-entry-policy)
- [§2 Cluster D-02 — Room status ownership and front_office_status](#2-cluster-d-02--room-status-ownership-and-front_office_status)
- [§3 Cluster D-03 — Daily assignments, shifts, business date, workload](#3-cluster-d-03--daily-assignments-shifts-business-date-workload)
- [§4 Cluster D-04 — Person discrepancies](#4-cluster-d-04--person-discrepancies)
- [§5 Cluster D-05 — Disconnected frontend features and missing endpoints](#5-cluster-d-05--disconnected-frontend-features-and-missing-endpoints)
- [§6 Cluster D-06 — Supply-request approval workflow](#6-cluster-d-06--supply-request-approval-workflow)
- [§7 Cluster D-07 — Assignment model](#7-cluster-d-07--assignment-model)
- [§8 Cluster D-08 — Legacy room_status disposition](#8-cluster-d-08--legacy-room_status-disposition)

---

## 1. Cluster D-01 — Room lifecycle and re-entry policy

**Business question.** What is the authoritative room-condition vocabulary, who may move rooms through it, what is the authorized re-entry path into service, and how do availability blocks (OOO/OOS) interact with condition? Findings: HK-006, HK-013–HK-016, HK-019, HK-020, HK-021, HK-007/HK-008 (who may approve).

### Current state (labeled)

- **F** — `rooms.housekeeping_status` default `'READY'`; `rooms.room_status` default `'AVAILABLE'`; `rooms.front_office_status` default `'CLEAN'`; composite availability index `(hotel_id, room_status, front_office_status)` exists (`schema.prisma:11174–11205`).
- **F** — FO check-in gate: assignable = housekeeping state ∈ {CLEAN, INSPECTED, READY}; OCCUPIED block; OUT_OF_SERVICE (FO flag) blocks; PICKUP/null/undefined not assignable (`room.aggregate.ts:30` gate; pinned by `room-readiness.spec.ts:43–66` — the only regression-protected part of this domain).
- **F** — Exactly two HK-relevant condition writers exist: checkout marks DIRTY (conditioned; `check-out.handler.ts:470–481`, unpinned by tests) and HK inspection approval writes INSPECTED (rooms + legacy `room_status` mirror, each failure-swallowed via `.catch(() => {})`, `housekeeping.service.ts:560–680`). No production writer for CLEAN, OOO/OOS, or `front_office_status` anywhere in the API (Phase 1 writer census, re-verified).
- **F** — Vocabulary fragmentation: shared constants define aspirational states (`CLEANING_IN_PROGRESS`, `RECLEAN_REQUIRED`, `TURNDOWN_PENDING`) and a full transition matrix (`housekeeping.constants.ts:10–43, 158–170`) that is not the implemented column vocabulary; the default status-flow config (served from `property_config` rows when they exist, otherwise these built-in defaults; `housekeeping.service.ts:937–947`) uses yet another vocabulary (VACANT_DIRTY / IN_PROGRESS / OCCUPIED_*), which matches neither the columns nor is it read by any transition logic (HK-021).
- **F** — OOO/OOS exist in vocabulary/UI and a maintenance table family, read-forked across ≥5 literal variants; `'OOO'` cannot pass the FO RoomStatus value object (HK-019/HK-020). `room_status_config` models a legacy 4-state display order (inspected/clean/pickup/dirty, `schema.prisma:10976–10988`).
- **I** — "approve-only re-entry" has been the de-facto behavior: it is the only implemented path; the product owner has now *approved* it as policy (HKD-02).
- **U** — Whether properties require a rush-clean "no-inspection" exception (e.g., early arrival); no requirement evidence exists.

### Policy options and evaluation (re-entry)

1. **Approve-only (INSPECTED qualifies; owner-approved).** Pros: single implemented producer guarantees "entered-service rooms passed inspection"; gate + tests already consistent; preserves supervisor accountability. Cons: supervisor absence can bottleneck arrivals (mitigation: PICKUP flow exists at task level today but stays non-assignable at the gate — accepted consequence on record); silent mirror-write failure path must be made fail-visible (Phase 3+).
2. **CLEAN also re-enters.** Cons: requires a CLEAN writer that does not exist; removes the only supervisor control point; contradicts the implemented intent. Not chosen.
3. **Property-configurable mode.** Cons: adds config semantics before a single consistent lifecycle exists; rejected for this phase — noted as a possible later evolution under the (config) surface that already exists (`hk_status_flow_config`).

### HKD records

**HKD-01 — Authoritative condition vocabulary.** **Status: PROPOSED** (owner ratification at ledger review).
- Policy: authoritative room-condition values are the implemented, gate-consumed set: `DIRTY`, `CLEAN`, `INSPECTED`, plus legacy defaults `READY` (synth/default) and `PICKUP` (task-level synthesis; not gate-assignable). Task-progress states (`IN_PROGRESS`-equivalents), `RECLEAN_REQUIRED`, combined prefixes (`OCCUPIED_DIRTY`…) are **task-level or derived display concepts, not room-column values**.
- Evidence: columns + gate + read-paths as above; constants/config matrices become *spec references only* until Phase 3 reconciles them (no changes now).
- Phase 3 consequence: Final Domain Specification defines one vocabulary; constants file alignment is an implementation-phase item, not schema work.

**HKD-02 — Re-entry authorization.** **Status: APPROVED.** Owner, 2026-10-20 ("Approve-only").
- Rule: the only authorized transition from unreleasable condition (DIRTY) back into service is HK inspection approval (the approve/reject surface, `housekeeping.service.ts:560–680`) writing INSPECTED. Front desk, reservations, and the legacy assignment path may never self-declare readiness. Legacy READY remains gate-accepted (compatibility; HKD-05). No schema/API change in this phase.
- Invariants: (a) INSPECTED is terminal-upward — no automatic downgrade; (b) reverse movement (reject inspection) sends the room to a *failed state for cleanliness* (DIRTY-class) — see HKD-03; (c) inspection approval must become fail-visible (Phase 3+ replaces `.catch(()=>{})` swallowing — HK-006/HK-008; *decision*: swallowed failures are not an acceptable design under the approved rule, implementation deferred).
- Roles (proposal cabled to decision, subject to security ratification OQ-12): attendants initiate task completion; HK supervisors (housekeeping_inspection audience) approve/reject; FO front desk consumes gate only; no cleaning staff approve path; admins/supervisors may reassign (ties to HKD-24/HKD-30).

**HKD-03 — Inspection failure / rejection loop.** **Status: PROPOSED** (engineering).
- Current: reject writes `rooms.housekeeping_status` `REJECTED`? — no: verified path writes the task to rejected/audit rows and leaves room state as written pre-inspection (Phase 1: approve/reject at `:560–680`; room write occurs on approve side).
- Policy proposal: rejection (i) leaves the room NOT assignable (DIRTY-class), (ii) creates/targets a re-clean task (manual today; auto-create as Phase 3 requirement), (iii) audit-logs inspector, reason, room, outcome (identity persistence is the HK-008 defect — Phase 3+ must attach authenticated inspector id, not client-supplied). Invariant: a room must never be saleable while an inspection is REJECTED/pending-resolution.
- Failure cases to address in Phase 3: repeated rejection ping-pong (no auto-escalation — acknowledged; escalation owner = HK supervisor), reject-on-occupied-room (FO busy FO transitions — must reconcile with HKD-31 ownership).

**HKD-04 — OOO / OOS availability-block ownership.** **Status: PENDING_OWNER_DECISION.** Queue OQ-03.
- Facts: no writer anywhere (HK-020); UI+vocabulary abound; engineering-maintenance module screens exist (`apps/web/app/(dashboard)/engineering-maintenance/`).
- Options: (a) maintenance domain owns OOO/OOS lifecycle, FO/HK consume (recommend; matches module existence and HK-020's "no lifecycle owner" need); (b) HK owns placement for room-cleaning-blocking needs (OOO *from* HK), FO owns OOS (dual-writer, needs matrix — heavier); (c) defer until maintenance module's scope is confirmed.
- Consequences: whichever is chosen, HK's *read* behavior (keep-out of OOO/OOS rooms) is unaffected; only the writer changes. No schema change in Phase 2.
- Owner question (OQ-03): which module may place a room OOO/OOS, and who authorizes (role) — plus whether HK can *request* a block.

**HKD-05 — READY semantics.** **Status: PROPOSED** (engineering; gate-pinned).
- Facts: `READY` is the column default and gate-accepted; tests pin READY assignable (`room-readiness.spec.ts:43–44`).
- Policy: READY is a legacy/default equivalence state ("clean-enough; inspection history unknown"), NOT an inspection outcome; the approved lifecycle (HKD-02) may write it only by column default/new rooms — never by inspection logic. Phase 3 spec must document this explicitly so `READY` ≠ `INSPECTED` reading drift (HK-014's promised INSPECTED-then-READY world is not adopted).

### Cluster dependencies
HKD-02 constrains HKD-03 (rejection stays DIRTY-class), HKD-09 (generation must not reset inspected/completed tasks — see D-03), HKD-27 (dashboard truth includes inspected counts), HKD-31/32 (which columns carry the approved policy). HKD-04's owner outcome must remain consistent with HKD-31 (availability axis authority).

---

## 2. Cluster D-02 — Room status ownership and `front_office_status`

**Business question.** Is `front_office_status` an authoritative business concept, a derived value, a mirror, or obsolete? Who owns what, who may transition, and how do conflicts/staleness resolve? Findings: HK-015, HK-019, HK-028/HK-029, HK-016.

### Current state (labeled)

- **F** — `rooms.front_office_status` has **no production writer in the entire API** (Phase 1 census; re-verified). Column default `'CLEAN'`; part of the availability composite index; FO gate reads it as an OUT_OF_SERVICE blocker; readiness reader *falls back* to it when `housekeeping_status` is null (`room-readiness.spec.ts:112–128`).
- **F** — The FO discrepancy comparator reads `rooms.front_office_status` and compares it against **housekeeping-state values** (`HK_VACANT_STATES = {CLEAN, INSPECTED, PICKUP, READY}`, `get-discrepancies.handler.ts:15,60,70`) — i.e., one live consumer assumes the FO column carries HK-style condition values. This is direct evidence that the column's *meaning is contested in code itself* (HK-028/HK-029 corroboration).
- **F** — `room_status` mirror table carries `housekeeping_status`, `front_office_status`, **and `dnd_status`** (`schema.prisma:10959–10971`); `dnd_status` has **zero consumers across apps** (grep-verified 2026-10-20).
- **F** — `room_status_config` (4-state display order) has no HK consumer; `room_status_history` exists as a generic change log.
- **I** — Because `housekeeping_status` is non-null-by-default on `rooms`, the "fallback when housekeeping absent" path is practically dead in normal data.
- **U** — Whether any future FO concept (e.g., "show the pick-up flag to front desk") is planned — a product-front-office question, not answerable from this module.

### Options

1. **Non-authoritative compatibility column (recommended).** HK spec treats it as: not authoritative; no HK writer (existing zero-writer state is therefore *policy*, not an accident to fix); fallback reads must be retired by Phase 4 implementation (they cannot trigger under the non-null invariant, and ambiguity is poison); any *future* producer/semantic belongs to the Front Office domain decision, not to HK. Pros: zero risk, honest about usage. Cons: comparator dependency (below).
2. **Repurpose as FO availability flag.** Gives OOO/OOS/OOS-clean/… a home; but invents a concept no requirement evidence supports, and it would collide with comparator assumptions. Rejected.
3. **Authoritative dual-condition field.** Treats the comparator's assumption as *the intended design* (FO column mirrors HK state). Pros: legitimizes HK_STATE_MISMATCH reading. Cons: contradicts the column NAME, duplicates `housekeeping_status` (HK-013 split-brain), and creates a second differential to mismanage — rejected.
4. **Obsolete → candidate for later DB retirement.** True but premature here: retirement requires the mirror/census work (D-08) and is out of this phase.

### HKD records

**HKD-06 — `front_office_status` semantic role.** **Status: PROPOSED** (engineering + FO-domain awareness; owner ratification at ledger review; queue OQ-05 trivially grouped).
- Policy: (i) HK must not read it as condition authority; (ii) HK must not write it; (iii) the FO discrepancy comparator's `HK_STATE_MISMATCH` leg must be re-scoped to read `housekeeping_status` *plus* `room_status` leg in the FO-owned flow — flagged as a cross-module spec change (HKD-18 depends); (iv) the read-fallback path is designated for removal in the implementation phase, with the explicit note that per non-null defaults it is currently unreachable for normal data.
- **Comparator caveat**: adopting this policy *before* re-scoping the FO comparator would transiently change HK_STATE_MISMATCH sensitivity. Recorded as a dependency in 04 (HKD-06 → HKD-18) so Phase 3 orders both.
- Cross-module impact: FO gate's OUT_OF_SERVICE blocker read stays (it is a *distinguishing differentiator* not a condition read); FO product decision needed only if they want the *concept*.

**HKD-07 — `dnd_status` (mirror table).** **Status: DEFERRED.** Zero consumers across apps (F). Policy: out of HK domain scope going forward; retirement is a mirror-family item (HKD-32) requiring the consumer census. No writes/read contracts declared in this phase.

**HKD-08 — `room_status_config` (4-state ordering legacy).** **Status: DEFERRED.** No HK consumer (F). Belongs to FO display-config legacy; candidate for retirement evidence gathering with HKD-32. Not part of the HK final domain spec unless FO claims it.

---

## 3. Cluster D-03 — Daily assignments, shifts, business date, workload

**Business question.** What does "daily generation" mean, what are shifts/business date, how is work preserved/handed over, and how do workload and cadence operate? Findings: HK-004, HK-009, HK-031, HK-036, HK-005 (collision abort).

### Current state (labeled)

- **F** — Model: `housekeeping_tasks` has **no business_date and no shift columns** (`schema.prisma:5759–5779`); tasks carry room_number, section, task_type, priority (`'normal'` default), status (`'pending'` default), assigned_to, start/completed timestamps, notes.
- **F** — Generation: `generateDailyTasks` resets **all** non-guest tasks to pending — including inspected/completed ones — records **no audit entry** for this reset, skips rooms with any existing task, aborts the entire run on a single `(hotel_id, task_id)` unique-key collision (manual endpoint only; no cron/Temporal/event trigger — Phase 1 verified).
- **F** — Task ID collision shape: service id-minting collides across rooms/day (HK-005 + HK-009) because ids are generated per-run while the unique key is `(hotel_id, task_id)`.
- **F** — Shifts absent in model/domain (HK-031 — store strings, no semantics anywhere). Workload/credit formulas absent anywhere. SLA config exists as **display-only estimates** consumed in `DashboardSection/DailyAssignments/GuestRequests` for "Est. complete" labels; **no overdue computation exists** in API (grep `slaMinutes` verified within UI-estimates + config endpoints only).
- **F** — True per-date work models exist next door as precedent: `room_turndown` (unique per room+hotel+service_date, `schema.prisma:11023`) and FO `room_discrepancies` (unique per room+hotel+business_date+type, migration 20260829_s1).
- **I** — "Checkout clean" should be a checkout-day-of work item; "ensure" pattern must never interact with in-flight guest services (guest-request tasks are exempt from generation, F — reset excludes guest tasks).
- **U** — Whether properties operate HK day/night shifts needing system-level splits (owner/ops); workload credit formula (ops).

### Options for the generation operation

1. **Ensure-missing (owner-approved).** Generate only tasks that do not exist yet for the effective date; never touch completed/in-progress/inspected; duplicate generation idempotent (no-op). Pros: matches the approved policy; preserves supervisor-approved progress; fits existing per-date precedents; makes the destructive reset a policy violation rather than designed behavior. Cons: requires Phase 3+ formalizing the effective-date key (HKD-11) to define "already exists" — recorded, not invented here.
2. **Refresh working list.** Preserves completed but resets unfinished pending work with audit. Rejected: unnecessary destructive behavior when ensure-missing is strictly safer; no operational requirement evidence for "reset unpicked tasks daily".
3. **Keep current reset-all.** Explicitly rejected by owner (superseded destructive behavior; remains a Phase-1 verified fact of current code until the lifecycle phases change it — no implementation in this phase).

### HKD records

**HKD-09 — Generation operation = ensure-missing.** **Status: APPROVED.** Owner, 2026-10-20 ("Ensure-missing").
- Rule: daily generation creates missing daily work for the effective date only; completed, in-progress, and inspected tasks must **never** be reset; repeated/concurrent generation must be safe (idempotent); existing tasks are skipped, not duplicated. No implementation in this phase (Phase 3/4 own it); the rule is the direct inverse of the behaviors recorded as HK-004/HK-009.
- Authorization: trigger authority = HK supervisor (and system admin); automation via scheduler/Temporal is *not* approved here — it remains an enhancement candidate requiring its own approval (HK-034 analog; `POST /housekeeping/tasks/generate-daily` route remains today's sole trigger).
- Failure cases for the future spec: (i) generation while a room has in-flight guest-request activity (exempt); (ii) generation after date rollover on unfinished prior-day work (see HKD-15); (iii) unique-key conflicts (explicit skip-with-count policy; no mid-loop aborts; no audit-free state changes).

**HKD-10 — Shift concept in scope.** **Status: PENDING_OWNER_DECISION.** Queue OQ-06.
- Facts: zero shift semantics exist anywhere (F). Options: (a) out of scope for the Housekeeping domain spec (recommended; risk: speculative domain invention); (b) in scope as a first-class shift attribute on tasks/assignments (requires ops model: shift names, hours, property-local time).
- Consequence of (a): a later scope-adding decision is a normal change; out-of-scope now avoids carrying unusable semantics. Consequence of (b): Phase 3 must define shift as a first-class concept; no schema change here either way.

**HKD-11 — Business-date attribution.** **Status: PROPOSED** (engineering; derived from HKD-09).
- Policy: "daily" work requires an attribution key. Proposal (consistent with existing per-date precedents): effective business date on generation, key `(hotel_id, business_date, room_number, task_type)`-class uniqueness for daily work — exact key composition is Phase 3+ design, **not** a mandate here; no schema change now.
- Consequences: enables idempotent ensure (HKD-09) and preserves room-lifetime history (HK-004's "once-per-room-lifetime" flaw dies with this); guest-request/critical tasks stay outside the "daily" identity, per current generation exemption.

**HKD-12 — Workload / credit policy.** **Status: DEFERRED.** No formula, no precedent, no requirement doc anywhere (F). SLA config's `slaMinutes` is display-only (F) — recorded as a **display-only policy** in Phase 3. Queue OQ-07 (ops: rooms-per-attendant target, credit values, min/max floor/ceiling) owns the formula when/if adopted. Nothing in the domain spec may claim credits without that approval.

**HKD-13 — Task preservation & concurrency invariants (derivation of HKD-09).** **Status: APPROVED (derived).** Owner-approved rule implies, verbatim: never reset completed/in-progress/inspected tasks; duplicate generation = no-op; per-generation run is one transactional batch; skip-on-conflict replaces abort-on-conflict; every state-changing generation action preserves an audit reference. (Counted as owner-derived; see 01 §6.)

**HKD-14 — Checkout-triggered vs daily work.** **Status: PROPOSED** (engineering; ratified by HKD-02/HKD-09 interaction).
- Policy: checkout marks DIRTY (approved existing behavior, HKD-02 axis); generation should create a checkout-adjacent cleaning task only for rooms lacking active non-guest work that corresponds to the DIRTY condition; departure-day tasks are distinguished from stayover tasks by trigger source, not by duplicate task creation. Exact task-type naming/merge rules are Phase 3+ design detail.
- Rationale: prevents double-assignment of the same physical job across the daily and checkout channels without inventing new states.

**HKD-15 — Unfinished work across days / handover.** **Status: PROPOSED** (engineering; NO auto-transfer).
- Policy: unfinished, still-open tasks **carry forward** with their original assignment; no automatic transfer to a new employee (no policy basis; avoids unapproved mutation of staff attribution); reassignment is an explicit, audited human act (ties to HKD-24/HKD-30). Handover visibility = by effective-date view, not by shift (shifts undeclared).
- Queue OQ-08 confirms whether ops want an auto-bucket rule later (optional evolution; documented).

---

## 4. Cluster D-04 — Person discrepancies

**Business question.** Is person discrepancy an approved HK capability, who owns occupancy authority, and what is the record lifecycle? Findings: HK-003 (+ ratified `person_discrepancies` declaration), HK-026/HK-030 adjacent.

### Current state (labeled)

- **F** — TWO parallel stacks exist for the same capability:
  (i) **Front Office CQRS stack — complete**: `room_discrepancies` table EXISTS in migrations (`packages/db/migrations/20260829_s1_room_discrepancies/migration.sql:5–28`; also in the C03 baseline capture `:15323`), schema model `17138–17156`, typed discrepancies (PHANTOM_OCCUPANCY / HK_STATE_MISMATCH / UNRECORDED_OCCUPANCY), upsert with business-date uniqueness, resolve endpoint requiring a user (`resolve-discrepancy.handler.ts:30`), resolved events, permission-gated controller routes (`front-office.controller.ts:49–69`; Phase 1's FO UI permission-gated row).
  (ii) **HK-tagged CQRS stack — broken end-to-end**: `person_discrepancies` never created (ratified, migration grep re-confirmed: zero matches in migrations), `HK-003` payload mismatches (`personCount` vs `hkPersonCount`), read route missing, feature mounted nowhere.
- **F** — The FO comparator does **not** count people: it reconciles *room-level occupancy truth* (`rooms.room_status` OCCUPIED set vs in-house reservations vs a vacant-state set read from `front_office_status`). "Person discrepancy" today = occupancy reconciliation, not headcount.
- **F** — No headcount/true person-count mechanism exists anywhere in the API (no source of "people observed in room").
- **I** — The authoritative occupancy source is the Reservations/FO in-house state (used so by the FO comparator).
- **U** — Whether any property's operational/legal workflow requires housekeeping-side *headcount* reporting at all (owner evidence; the capability's *name* suggests it, the *implementation* (both stacks) contradicts it).

### Capability decision (owner-approved)

**HKD-16 — Adopt the FO stack; retire the HK-tagged CQRS stack by disposition.** **Status: APPROVED.** Owner, 2026-10-20 ("Adopt FO stack").
- Rule: the person-discrepancy capability is owned by the Front Office domain; `room_discrepancies` is its record. The HK CQRS discrepancy stack (unmounted; mismatched payloads; dead table) is dispositionally retired — actual code retirement happens in later lifecycle phases' own gates, *no code change now*. HK UI consumes FO data if/when a need is shown (D-05 HKD-25 covers the HK-side pointers).
- Consequences: no schema change this phase; `person_discrepancies` stays a ratified known-defect record (its "create table" branch is now moot; the declaration's disposition note should be revisited in the DB-integrity workstream, not here); CQRS structural reuse is proven done right side.
- Guardrail (per evidence honesty): this approval does NOT establish that any true *headcount* capability is a requirement (see OQ-09) — it selects which existing implementation is authoritative for what exists today.

**HKD-17 — Occupancy authority.** **Status: APPROVED (derived from HKD-16).** Owner's stack adoption implies the FO-side reconciliation contract: (i) in-house Reservations + FO `room_status` are the authoritative occupancy facts; (ii) HK contributes condition/occupancy observations via its own approved channels only — never by writing occupancy truth; (iii) the comparator's vacant-state set must be re-anchored to `housekeeping_status` (HKD-06 dependency) during Phase 3, because the current FO-column read is meaning-contested (F above).

**HKD-18 — Record lifecycle & retention.** **Status: PROPOSED** (engineering; the rules required by the brief).
- Lifecycle (proposed): rows are OPEN on detection (upserted), carry `business_date`, and end in RESOLVED (requires a resolver — already enforced in code, F) or are voided/approved-by policy; reopen-on-recompute semantics exist today (upsert resets RESOLVED→OPEN on state change, F, `discrepancy.repository.ts:50`) and are retained. Retention: keep dated rows (audit-grade; matches mandated "what information must be retained").
- Who may resolve: FO front desk/user with the discrepancy permission (implemented; F) — HK supervisors consume but do not resolve under the adopted contract.
- Phase 3 consequence: the domain spec defines exactly these states and the reopen-on-recompute rule (documented today-verifiable, not re-designed in Phase 2).

---

## 5. Cluster D-05 — Disconnected frontend features and missing endpoints

**Business question.** For each capability the HTML calls but cannot reach: required / inconsistent-integration / optional / retire / unresolved — plus requirements if required. Findings: HK-001, HK-002, HK-022, HK-023, HK-024, HK-029, HK-030, HK-033, HK-034; register Annex A.

Default disposition policy (owner-approved): **case-by-case** (HKD-19) — no group is presumed necessary or removable; each entry below is a separate disposition with its own evidence requirement.

### HKD records

**HKD-19 — Disposition default = case-by-case review.** **Status: APPROVED.** Owner, 2026-10-20 ("Case-by-case").
- Rule: every disconnected capability group gets its own reviewed disposition before build or removal; no blanket presumption either way. Queue OQ-02 owns the batch.

**HKD-20 — Guest requests (HK↔room-service).** **Status: PENDING_OWNER_DECISION.** Queue OQ-04.
- Facts: requests UI exists in both modules coupled by a string protocol with a corrupted-history side effect (HK-030); POST-store calls to `/housekeeping/requests…` are dead (HK-001); `/room-service/requests*` routes are live (Annex A) and room-service module owns request semantics today.
- Recommendation: room-service domain is authoritative for request lifecycle; HK consumes/creates through it; HK-side string protocol retired in Phase 3 spec. Owner confirms priority + SLA expectations (ops).
- Requirements if approved as capability: authoritative source = room_service_orders/requests tables; persistence+audit on both record sides; SLA display via HKD-12-adjacent config; ui-impact = guest-request tab works end-to-end vs today's persistence-free illusion.
- Consequence if unavailable: quotes fetched work by accident of existing live `GET` routes only where implemented — the write path remains fake (operator writes requests that vanish).

**HKD-21 — Linen management (HK-side writes).** **Status: PENDING_OWNER_DECISION.** Queue OQ-10.
- Facts: reads live (synthesized location splits with a fixed 60/25/10/5 ratio — HK-035); all linen write/read flows dead; a *fabricated client-side refill loop* writes only to store state (HK-002); seeded real par data exists (`linen_inventory` per-room pars + `packages/cli` seed).
- Recommendation: REQUIRED capability candidate (core hotel ops) — real persistence. Owner confirms: full scope (issue/retire/return/laundry-loop) vs initial minimum; par-location model alignment (HKD-35 note in analysis: tech-inventory alignment is ripe — the *decision* is persistence of what exists).
- Consequence if unavailable: open/checkout dressing packets, housekeeping maintenance counts, and supervisors all operate on imaginary inventory counts (HK-002's core harm).

**HKD-22 — Laundry.** **Status: PENDING_OWNER_DECISION.** Queue OQ-10 (grouped with linen).
- Facts: full UI screens; zero backend routes for the whole flow (HK-001); source types (rooms/restaurant/spa/gym/pool/uniforms/public-areas) show a *group* capability design.
- Requirement: owner decides in-house vs outsourced; if in-house, laundry becomes a required capability (lifecycle states exist aspirationally); if outsourced, likely retire-down to a contract/record minimum.
- Evidence needed: property operational model (laundry house vs vendor).

**HKD-23 — HK "teams."** **Status: PENDING_OWNER_DECISION.** Queue OQ-11.
- Facts: `/housekeeping/teams` dead (HK-001); staffing model in code is attendants+sections (`housekeeping_attendants` unique per hotel+employee, schema `5708–5712`) with a supervisor flag carried in `hk_room_assignments.type` (`housekeeping.service.ts:501+, 837/850`); no teams table in schema (grep: model list has no teams); provisioning has no write path (HK-024).
- Recommendation: retire the teams concept in favor of attendants+sections (single staffing model); owner ratifies (this also folds HKD-03 supervisor identity clarity). No code change in this phase.

**HKD-24 — Task move-room.** **Status: PROPOSED** (engineering; small).
- Facts: service method exists but is unrouted (Phase 1); UI writes `POST /housekeeping/room-assignments/move-room` dead (store `:814`); operational need is real (room becomes blocked mid-clean; work must relocate).
- Disposition: approve as an existing-capability integration item (route + wire; no redesign); **requires** the invariant that move-room maintains the approved re-entry rule (HKD-02) and cannot move an in-flight task to a room violating OOO/OOS reads. Owner-queue follow-up unnecessary; ratification happens at ledger review with D-07.

**HKD-25 — Lost-found claim pointer + HK-references in the HK UI.** **Status: PROPOSED** (engineering; UI delegation).
- Facts: `POST /lost-found/items/:id/claim` dead (HK-001); lost-found module exists as its own dashboard (own pages, its own store); HK-side reference/claim affordances are duplicates of the module owner feature.
- Disposition: remove duplicate affordances from the HK UI (Phase 9+ UI work) and rely on lost-found module data; no backend change. Norm: a module's dashboard should reference/deep-link, not re-implement.

**HKD-26 — Dead components (6 of 12, incl. `MinibarSection` crash-if-rendered).** **Status: PROPOSED** (engineering).
- Facts: HK-023 — six unnamed live; six dead, one crashes on render (verified P1: `MinibarSection` would crash if ever rendered, e.g., via a future tab restore).
- Disposition: obsolete → retirement candidates alongside Phase 9+ UI plan (they also invite the crash risk). Any true minibar/lost-found/deep-clean* capability that remains desired enters the case-by-case queue like HKD-21/22 rather than via resurrecting dead components.

**HKD-27 — Dashboard fabricated statistics → server-authoritative truth.** **Status: PROPOSED** (engineering; data-truthfulness norm).
- Facts: tab statistics are client-synthesized with fake affordances (HK-022); `GET /housekeeping/dashboard` exists unused (HK-034) and is server-side (real counters including inspected/rejected — HK-034's evidence row).
- Disposition: adopted policy = operational counters rendered anywhere must come from server data; Phase 3 spec records which counters are authoritative; unused endpoint is the ready candidate (or it is removed — ratify at ledger review). Whether "fabricated counters" are an error-type UX defect (HK-022) or have an ops case is queued (OQ-13, low).

---

## 6. Cluster D-06 — Supply-request approval workflow

**Business question.** Whose capability are HK supplies, what is the intended lifecycle, what states/policies are justified? Findings: HK-037 (approve dead-end), HK-026, Annex A (no UI consumer), inventory stack exists and is live.

### Current state (labeled)

- **F** — HK stack: full lifecycle surface (`VALID_TRANSITIONS`, 8 declared states) + numbering + audit + history tables (`housekeeping_supply_requests` family, `schema.prisma:16225–16395`); `approve()` rejects every reachable submit path (`:379–405`, HK-037); no UI consumer anywhere; the separate `inventory` supply-request stack exists and is UI-live (module glob: controller + approval + domain + application + validation services + events + DTOs + constants).
- **F** — Inventory stack's `/supply-requests` used by `apps/web/lib/inventory/...` (Annex A); own approval service exists (`supply-request-approval.service.ts`); warehouse structuring (`warehouse_id` migration on supply request — separate lineage, out of scope).
- **I** — Operational need for *HK requesting supplies* exists in the real world (housekeeping wants minibar/linen/(training) stock), but that need does NOT dictate owning a second lifecycle.

### Options and decision

1. **Consolidate into inventory/procurement domain (owner-approved).** HK requests supplies through the inventory domain; the HK-tagged supply stack is dispositionally retired (actual retirement in later phases' own gates). Pros: single lifecycle owner; the broken approval chain (HK-037) becomes an inventory-domain requirement rather than an HK-domain fix to duplicate; HK-facing view added later per D-05 case-by-case. Cons: cross-domain UI/permission routing needed later.
2. **Keep HK-owned lifecycle.** Fix theApprove dead-end, build HK UI, accept duplicate-domain risk. Rejected by owner.
3. **Both stay.** Rejected (two lifecycles for one capability contradicting the ledger's single-authority norm).

**HKD-28 — Domain ownership = inventory/procurement.** **Status: APPROVED.** Owner, 2026-10-20 ("Inventory owns it").
- Rule: housekeeping supply requests are ordered through the inventory domain's request capability; HK must not own a second supply-request lifecycle. Locus of any future HK-facing view (Phases 9+ UI) under disposition queue OQ-14 alignment with HKD-19 case-by-case.
- Consequences recorded: HKD-29 below; `housekeeping_supply_request_*` tables/markers remain a disposition backlog item (retirement candidates); migration-baseline declaration impact not introduced here.

**HKD-29 — Lifecycle states policy.** **Status: DEFERRED.** With ownership decided, the states/transition/quantity/approval-level policies belong to the *inventory domain's* spec (Whose approval levels? partially-fulfilled semantics? multi-level approvals at `housekeeping_supply_requests` rows `:383–388`? — all those details are inventory-side questions now). Deferred in *this* ledger; recorded as an input the inventory module's domain spec must accept (not an HK Phase 2 obligation).

---

## 7. Cluster D-07 — Assignment model

**Business question.** Which representation is authoritative for staff-to-room work: room-embedded shared JSON fields, `hk_room_assignments` cards, or neither? Findings: HK-011, HK-024, HK-046-adjacent; generateDailyTasks pushing `assigned_to` (`:197`).

### Current state (labeled)

- **F** — THREE loci actually exist today: (i) `housekeeping_tasks.assigned_to` (per task); (ii) `hk_room_assignments` rows that are **assignment CARDS**, each embedding a rooms JSON array (`rooms` Json default `"[]"`, `type` VarChar(30) distinguishing `type='supervisor'` via service lines `:214, 501–557, 837/850`), and CRUD routes both attendant & supervisor (`housekeeping.controller.ts:258–273, 317–332`); (iii) store state (`roomAssignments`).
 It is NOT two representations; it's cards-with-JSON-payload vs task-attribution, plus reassign writing a supervisor id over an attendant's task assignment (HK-011).
- **F** — No assignment entity has: identity-per-room-work, business-date attribution, duplicated-room prevention, lock semantics, or per-room audit trail. `room_status_config`/`room_status_history` patterns exist elsewhere but not here.
- **F** — Consumers today: store fetch/POST/PUT/DELETE of `/housekeeping/room-assignments` (live), supervisor routes (live), generation writes only `assigned_to`.
- **I** — The operational requirement is a **day-plan**: attendants get a set of rooms for a date, with supervisor overlays; work state of each room lives on the task rows, not the card.

### Options

1. **Tasks own work-state; cards own day-plan (recommended).** `housekeeping_tasks.assigned_to` remains the authoritative per-room-work ownership; `hk_room_assignments` are planning cards (attendant-day assignment sheets) — identity, handover-tool data, NOT per-room work-state storage. Pros: no invented model; big reduction in the HK-011 overload ambiguity; reassign semantics become task-based (fix-by-spec of HK-011). Cons: card JSON stays semi-structured (needs Phase 3 shape decision if cards persist).
2. **Relational per-room assignment ledger (new rows per room+attendant+date).** Strong queryability/history — but the per-room ledger would duplicate task-level state, risks divergent truth between tasks and ledger, and any new table work belongs to later schema phases (not planned or approved here). Rejected for now; recorded as a possible future evolution if audit analytics require it.
3. **Cards only (dropping task attribution)**. Inverts truth; breaks current consumers (generation writes `assigned_to` on tasks); cross-module FO worklist unaffected either way. Rejected.

**HKD-30 — Authoritative representation.** **Status: PROPOSED** (engineering; owner ratification at ledger review; queue OQ-15).
- Rule (proposed): per-room work ownership is authoritative on `housekeeping_tasks.assigned_to`; `hk_room_assignments` are day-plan cards (including supervisor flag), and their JSON rooms array is a *plan*, never work-state; reassignInspection writes supervisor ids only on supervisor-role cards, never onto attendants' task attribution (Phase 3 re-spec of HK-011).
- Consumers affected by any future change (recorded): HK store assignments fetch/create/update/delete; `DailyAssignments`/`Inspection` UI role listing paths; the dead `move-room` route (HKD-24 wires it into tasks, not cards).
- Migration/compatibility implications of not changing anything now: zero code change; only spec-shape consequences; any data migration (card-room JSON → structured) would be an explicit later-phase decision with its own gate (recorded, not planned).

---

## 8. Cluster D-08 — Legacy `room_status` disposition

**Business question.** Is the legacy `room_status` field/table authoritative anywhere; what is its read/write policy; what's required before any retirement? Findings: HK-013, HK-014, HK-019; the Phase 1 writer census (`05 §7`).

### Current state (labeled)

- **F** — Three "status" fields co-exist with **different default vocabularies**: `rooms.room_status` default `'AVAILABLE'` (an *operational availability* axis, written by reservations room-move — F-25 real-Postgres-tested in `reservation-schema-contract.postgres.spec.ts:340–365` — and read by FO gate as OCCUPIED-block), `rooms.housekeeping_status` default `'READY'` (condition; HKD-02 axis), `rooms.front_office_status` default `'CLEAN'` (no writers; D-02).
- **F** — The `room_status` mirror table carries both HK and FO status columns plus `dnd_status` (zero app consumers; HKD-07), and is written by exactly: HK inspection approve (both `.catch(() => {})`-swallowed legs) and the reservations room-move contract (F-25). Readers: the FO discrepancy comparator reads the *rooms columns*, not this table; Phase 1 found the mirror consumed via service paths (HK-013's "split-brain between rooms and the legacy `room_status` mirror").
- **I** — The field's and table's single most dangerous property: two writers with divergent semantics (HK approval independent of reservation state) can silently disagree — exactly HK-013's registered risk.
- **U** — Full consumer census of the mirror table (`room_status` rows + `room_status_history` + `room_status_config` read paths) — required before any retirement decision; queued as evidence (OQ-20), not derivable from static reading alone with certainty.

### Disposition policy (proposed) — HKD-31, HKD-32

**HKD-31 — `rooms.room_status` (rooms-row column).** **Status: PROPOSED** (engineering; owner ratification).
- Rule: operational availability axis (VACANT/OCCUPIED/AVAILABLE-family + OOO/OOS-adjacent literals) — owned by the FO/reservations side (check-in/out, room-moves); HK is strictly read-only on this column (its approved behaviors already comply — checkout writes `housekeeping_status`, not `room_status`; F). Phase 3-documentation-level change only.
- Interaction note (must be preserved): the FO gate composes BOTH axes (availability non-OCCUPIED **and** condition ∈ {CLEAN, INSPECTED, READY}); the HK spec must define the composition, not merge them.

**HKD-32 — `room_status` mirror table (+ history + config).** **Status: BLOCKED_BY_EVIDENCE** (for retirement); interim policy PROPOSED.
- Interim policy (proposed): treat the mirror as a **compatibility mirror** (reads allowed where they exist; no NEW consumers; HK writes continue *only* through the approved lifecycle paths that already touch it — inspection approve and the reservations room-move contract — until Phase 3/4 replaces the swallowed-error legs with fail-visible behavior; HKD-02 invariant (c)).
- Mirror-family hard rule: `housekeeping_status` values written to the mirror must mirror `rooms.housekeeping_status` (single-source discipline, spec-level; whoever writes one writes both until separation is approved).
- Retirement: blocked pending (i) full consumer census (static + runtime data check; OQ-20), (ii) the reservations room-move F-25 contract's read dependence assessment, (iii) an explicit DB-phase decision (out of this phase's authority/mo ops). Record kept BLOCKED_BY_EVIDENCE rather than PROPOSED so Phase 3 cannot silently start writes/removals.

---

## 9. Cluster-by-cluster evidence trail (summary)

| Cluster | Load-bearing verified facts reused | Primary findings |
|---|---|---|
| D-01 | writer census; gate `room.aggregate.ts:30`; tests pinning gate; constants/config vocab vs columns | HK-006, HK-013–016, HK-019–021 |
| D-02 | zero-writer `front_office_status`; comparator reads it as HK-state (`get-discrepancies.handler.ts:15,60`); DND zero consumers; `room_status_config` legacy | HK-015, HK-019, HK-028/029 |
| D-03 | task model fields (`5759–5779`); generation reset/abort; per-date precedents (`room_turndown`, `room_discrepancies` uniqueness) | HK-004, HK-005, HK-009, HK-031, HK-036 |
| D-04 | `room_discrepancies` migration exists; HK `person_discrepancies` absent; comparator contract; resolve requires user | HK-003 (+ratified declaration) |
| D-05 | Annex A cross-walk; HK-001/002; seeds par data; dead components/crash | HK-001, HK-002, HK-022–024, HK-030 |
| D-06 | `:379–405` approve dead-end; inventory supply-request module files list (glob) | HK-037, HK-026 |
| D-07 | `hk_room_assignments` card shape (`5781–5794`); service card usage (`214,501–557`); task `assigned_to` | HK-011, HK-024 |
| D-08 | rooms defaults; mirror table columns incl DND; F-25 contract tests | HK-013, HK-014 |

No facts above are runtime-verified; they are source/inspection-verified (F labels) and marked accordingly, per the Phase 1 evidence discipline inherited.

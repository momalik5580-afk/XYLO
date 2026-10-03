# XYLO Availability Phase 5 — Final Domain Specification (Stage 3)

**Artifact:** `docs/availability/phase-5/03_FINAL_DOMAIN_SPECIFICATION.md` (artifact **03** of Phase 5)
**Preceded by:** `01_FORENSIC_AUDIT.md` (Stage 1 — forensic audit) · `02_BUSINESS_RULES_DECISIONS.md` (Stage 2 — Business Rules / Decisions, **CLOSED**)
**Followed by (not started):** Stage 4 — Implementation Plan, then Readiness Review, then Implementation.
**Status:** this document is the **single authoritative Phase 5 domain contract**. Stage 2 decisions are closed authority and are incorporated, not re-opened. Writing this file makes **zero** source, schema, migration, frontend, test, dependency, or configuration change, and performs no GitHub/remote operation.

---

## 1. Purpose

This document defines **what the XYLO availability domain means and what the system MUST do after Phase 5** (Other Consumers + Frontend Migration). It converts:

1. the ratified Phase 1–4 authority (preserved, never re-opened),
2. the closed Phase 5 Stage 2 business decisions and their 51 business rules,
3. every **Stage 3 Requirement / Input** stated by Stage 2 (blocker requirements, finding-level requirements, entry conditions, evidence requirements, guardrails, duty statements),

into **explicit, testable domain requirements, contracts, invariants, and acceptance conditions**, with complete traceability `F-xx → DS-xx → BR-5-xxx → requirement/input → specification section → acceptance condition`.

It is **not**: an implementation plan, a task list, a coding guide, a UI/UX design, a test report, or a generic requirements backlog. Stage 4 MUST be able to build the Implementation Plan from §36 **without inventing any business rule that belongs here**.

Stage 2 is **closed**: where this document and `02_BUSINESS_RULES_DECISIONS.md` §20 differ in wording, **the Stage 2 rule text governs**; this document expresses that same rule as contract structure and specification detail. Any genuine contradiction discovered later is documented and returned to the decision protocol (§3.4) — never silently resolved.

## 2. Scope

### 2.1 In scope (specified here)

Authority model; canonical read contracts (snapshot + matrix projection); the restriction evaluator contract (DS-01); UNRESOLVED/unresolved-capacity semantics; the availability state model; backend and frontend consumer contracts; frontend domain boundary; legacy source dispositions; the API domain contract; property/hotel isolation; restriction write ownership; dual-write/cutover rules; reconciliation, canonical logs and audit contracts; error/failure semantics; the environment/feature-flag contract; domain invariants; acceptance conditions; blocked/deferred register; Phase 4 carry-over dependencies; prohibited/deferred behaviors; complete traceability; Stage 4 inputs.

### 2.2 Out of scope (explicit, with status)

| Item | Status | Basis |
|---|---|---|
| Implementation plan, task IDs, sequencing beyond decided ordering gates, file-level modification plans, estimates | Stage 4 | stage boundary (brief §26) |
| UI/UX design (colors, layout, components, interaction) | Out of this stage | brief §27 |
| Schema changes, migrations, indexes, constraints, seeds, backfills | **Prohibited** | user constraint; Phase 4 deviation C (`17_…:70`); guardrail G-1 |
| Test execution / CI runs in this stage | Stage 4+ executed-evidence duty | BR-5-033/034 |
| Outbound channel/CRS push contract (B-6) | `DEFERRED` | P-19 (`13_…:65`); DS-07 |
| Multi-property contracts (B-7) | `DEFERRED` | TR-14.3 (P-15); DS-08 |
| Occupancy analytics contract (formula, realtime vs batch) | `DEFERRED` | P-20 (`13_…:67`); DS-06 |
| Legacy table deletion / migration cleanup | Phase 11 | P-16 (`13_…:731-735`) |
| Phase 4 open actions (commit delta, canonical cutover, cascade flag config, BLK-1/BLK-2) | Phase 4-owned | `17_…:92-99` (F-24) |
| Pickup ↔ assertion port shape (DEF-3/T-41) | Inherited, not re-decided | `16_RR_DECISIONS.md:28-38` |
| Hygiene: dead code (F-21), duplicate routes (F-25), doc drift (F-23) | Stage 4, gated **after** behavior-preserving work | audit §14 B-8; G-12 |
| Explicit restriction precedence ranking | `UNRESOLVED — EVIDENCE REQUIRED` (E-2) | Stage 2 §5 prohibition |

## 3. Authority Hierarchy

### 3.1 Tiers (highest first; a lower tier may never override a higher tier)

1. **User constraint / local workspace:** no schema changes for this phase family (AGENTS.md; Phase 4 deviation C); GitHub/remote is **not** authoritative; local workspace evidence wins over any remote material.
2. **Ratified Phase 1–2 contract** — `docs/enterprise/availability-phase1-2-contract-extract.md` (§4 snapshot contract, §5 assertion contract).
3. **Ratified Phase 3 rulings** — `availability-phase3-domain-specification.md`, decision sheet, Stage B ratification.
4. **Ratified Phase 4 corpus** — `11_TARGET_BUSINESS_RULES.md` (TR-*), `13_FINAL_DOMAIN_SPECIFICATION.md` (INV-*, §11/§18/§19), `10_DECISION_RESOLUTION.md` (D-*), `14_IMPLEMENTATION_PLAN.md` (T-*, §8/§11/§13), `15_READINESS_REVIEW.md`, `16_RR_DECISIONS.md`, `17_PHASE4_READINESS_REVIEW.md`.
5. **Phase 5 Stage 2 (CLOSED)** — `02_BUSINESS_RULES_DECISIONS.md`: 12 decision items, 51 business rules (BR-5-001…051), blocker register, evidence register, guardrails.
6. **Phase 5 Stage 1 (evidence only)** — `01_FORENSIC_AUDIT.md`: findings F-01…F-27, risks, inventories; evidence for deriving requirements, never a source of new policy.
7. **Existing tests and local implementation evidence** — behavioral evidence of what is true today (file:line).
8. **Legacy/product behavior** — lowest; may never by itself establish a rule contradicting tiers 1–5.

### 3.2 What this hierarchy forbids

- Re-opening a ratified Phase 1–4 rule or a Stage 2 decision because another option is easier (G-11, Stage 2 §5).
- Treating legacy behavior as authoritative because it exists (Stage 2 §3).
- Deriving a business rule from Stage 1 findings without a Stage 2 decision (Stage 1 records, never decides).

### 3.3 Stage 2 closure

Stage 2 statuses (`RESOLVED`, `PRESERVED FROM PRIOR PHASE`, `NEW DECISION`, `AMENDED DECISION`, `UNRESOLVED`, `BLOCKED`) carry into this document unchanged. `UNRESOLVED — EVIDENCE REQUIRED` items are carried as blocked specifications (§31), never defaulted into behavior.

### 3.4 Contradiction handling (no silent resolution)

If Stage 4, implementation evidence, or any later review surfaces a contradiction between this document, Stage 2, or Phase 1–4: **stop, record the contradiction with citations, and identify the exact decision requiring resolution** (Stage 2 §5 amendment protocol for prior authority; a new decision item otherwise). No unofficial business rule may be created to paper over a contradiction.

## 4. Source Documents

| # | Document | Role here |
|---|---|---|
| 1 | `docs/availability/phase-5/02_BUSINESS_RULES_DECISIONS.md` | Closed decisions; rules BR-5-001…051; blocker/evidence/guardrail registers; Stage 3 Requirements/Inputs source (§7 below) |
| 2 | `docs/availability/phase-5/01_FORENSIC_AUDIT.md` | Findings F-01…F-27, risks R-01…R-15, consumer/legacy/frontend inventories, compliance matrix, migration boundaries B-1…B-8 |
| 3 | `docs/availability/phase-4/13_FINAL_DOMAIN_SPECIFICATION.md` | Preserved domain authority: INV-1…INV-21, §1.2 out-of-scope, §3 fact ownership, §11 authority/data flow, §12 concurrency, §18 legacy boundaries, §19 reconciliation |
| 4 | `docs/availability/phase-4/14_IMPLEMENTATION_PLAN.md` | Preserved duties: T-38 (matrix rebuild/shape preservation `:563`), §8 discipline (`:813-840`), §11 rollout/flag sequence (`:923-941`), T-39 retirement precedent (`:566-568`), T-41/DEF-3 |
| 5 | `docs/availability/phase-4/15_READINESS_REVIEW.md`, `16_RR_DECISIONS.md` | Preserved readiness decisions; DEF-3 pickup port shape (`16_…:28-38`) |
| 6 | `docs/availability/phase-4/17_PHASE4_READINESS_REVIEW.md` | Deviations A–D, flag states, test baselines, `gba.consumers.cascade` ON-at-deploy (`:80`), Phase 4 open actions (`:92-99`) |
| 7 | `docs/availability/phase-4/11_TARGET_BUSINESS_RULES.md`, `10_DECISION_RESOLUTION.md` | TR-* rules; D-6/D-6a authority decisions (`10_…:87-125`, `:114`) |
| 8 | `docs/enterprise/availability-phase1-2-contract-extract.md` | Snapshot contract §4.1–4.5; assertion contract §5.1–5.8 |
| 9 | `docs/enterprise/availability-phase3-domain-specification.md` (+ decision sheet / Stage B ratification) | Fail-closed `:345`, `:403`; no release by deletion / Decision 12 `:141`; `CHECKED_OUT` `:180,:190`; status classification `:191` |
| 10 | Local implementation evidence (`apps/api/**`, `apps/web/**`, `packages/**`) | File:line facts used by Stage 1/2; behavior evidence only |

## 5. Phase 1–4 Rules Preserved

### 5.1 Preserved decision register (P-1…P-22 — carried verbatim from Stage 2 §4; NOT re-opened)

| ID | Preserved decision | Citation |
|---|---|---|
| P-1 | Snapshot is read-only; authority exposes exactly `GET /snapshot` + `GET /reconciliation`; writes go through the Phase 2 port. | extract `:57-61` (§4.1) |
| P-2 | Every source reports `RESOLVED`/`UNRESOLVED` and **declares itself unresolved rather than guessing**; restriction source has six dimensions (`closedToSell, closedToArrival, closedToDeparture, minLos, maxLos, sellLimit`). | extract `:63-76` (§4.2) |
| P-3 | **Fail-closed propagation:** unresolved/blocked ⇒ `physicalAvailable: 0`, `sellableAvailable: 0` (conservative floor, *not* zero-with-certainty), `bookingEligibility: 'UNKNOWN'`/`'BLOCKED'`; **a caller may not treat `UNKNOWN` as `ELIGIBLE`.** | extract `:78-95` (§4.3) |
| P-4 | Capacity arithmetic: all floors `max(0,…)`; **`sellLimit` only ever tightens** capacity; overage measured, not forbidden. | extract `:97-117` (§4.4) |
| P-5 | Assertion failure semantics: unresolved upstream ⇒ persisted `REJECTED` with `UNRESOLVED_CAPACITY`; **fail-closed is the default, not an error path** — "when capacity cannot be safely resolved, nothing commits." | extract `:217-232` (§5.7) |
| P-6 | Availability mutation is **synchronous and fail-closed** inside the reservation transaction; no outbox/saga may substitute. | Phase 3 spec `:345` |
| P-7 | Unresolved capacity ⇒ no new consuming commits; **"never assume zero or available."** | Phase 3 spec `:403` |
| P-8 | **Cancellation, not deletion, is the lifecycle operation**; *no availability release may be achieved by deleting a reservation* — release only through the exact active reservation-linked assertion; hard-delete only for terminal statuses (Decision 12). | Phase 3 spec `:141` |
| P-9 | **`CHECKED_OUT` does not release its assertion** (ratified invariant). | Phase 3 spec `:180,:190` |
| P-10 | Unrecognised/unmapped status is fail-closed (`UNRESOLVED` classification). | Phase 3 spec `:191` |
| P-11 | Exactly four fact kinds; **only their combination into sellable availability is single-sourced, and only by Availability.** | `11_TARGET_BUSINESS_RULES.md:16` (TR-1.1) |
| P-12 | **Availability is the sole source of the sellable number**; no second inventory engine; no independent "available" number anywhere (incl. rebuilt A3). | TR-1.2 (`:17`), TR-10.1 (`:122`), INV-18 (`13_…:610`), `10_…:114` |
| P-13 | Eligibility filters live in one place (TR-10.2); **stale-by-design prohibited** (TR-10.3); provenance + `UNRESOLVED` flagging are part of the contract (TR-10.4). | `11_…:123-125` |
| P-14 | **Stop sale = allotment-scoped selling permission**: never changes quantity, never releases; display remaining 0 while applied; evaluation order stop-sale-before-remaining; per hotel+date+category. | TR-7.1–7.5 (`11_…:89-93`), INV-17 (`13_…:609`) |
| P-15 | **Hotel isolation on every read/write incl. raw SQL**; no bare-id mutation; no cross-hotel availability aggregation; multi-property authorization does not exist and all rules are single-hotel. | TR-14.1–14.5 (`11_…:166-170`), INV-1 (`13_…:593`) |
| P-16 | Legacy GBA tables and legacy `availability` counters: retained, **read-only, never authoritative**; legacy deletion/migration = Phase 11. | TR-15.1 (`11_…:176`), `13_…:718-736` (§18) |
| P-17 | Old and new availability computations **do not coexist**: derived/relabelled views only (A3 rebuild = Option A; interim label `"derived view, not sellable availability"`). | TR-15.5 (`11_…:180`), D-6/D-6a (`10_…:87-125`), plan T-38 (`14_…:563`) |
| P-18 | A3 matrix **response shape is preserved**; its values come from the authority; availability numbers move in Phase 5 (A3 restriction writes ownership is decided by Stage 2 DS-04). | plan T-38 (`14_…:563`) |
| P-19 | External channel/CRS push contracts and multi-property contracts are out of scope (deferred). | `13_…:65` (§1.2) |
| P-20 | Analytics computation decisions (real-time vs batch) are out of scope (backlog). | `13_…:67` (§1.2) |
| P-21 | Rollout flags default OFF, are env `FEATURE_*`, follow the plan §11 activation sequence; **`gba.consumers.cascade` MUST be ON at deploy**; wash is blocked by Deviations A+B. | plan §11 (`14_…:923-941`), `17_…:80`, `17_…:64-72` |
| P-22 | Phase 4 test evidence: 178 API suites / 1470 pass / 1 skip / 6 fail with DB env; documented baselines API 2 suites / 6 tests and web 1 suite / 10 tests. | `17_…:45-47`, `17_…:57-60` |

### 5.2 Additional ratified rules invoked (preserved; listed so Phase 5 cannot drift from them)

| Rule | Statement (condensed) | Source |
|---|---|---|
| Phase 1 arithmetic | `physicalCapacity = max(0, physicalCount − unavailablePhysicalCount)`; `sellableCapacity = sellLimit === null ? capacityWithOverbooking : min(capacityWithOverbooking, max(0, sellLimit))`; `sellableAvailable = max(0, sellableCapacity − consumption)`; `overbookingUsed` measured, not forbidden | extract `:97-117` |
| Phase 1 source contract | 4 sources independently evaluated; unresolved carries `reasons` + `unresolvedSources`; restriction failure carries a reason | extract `:63-76` |
| Phase 2 DB invariants | balances `asserted_quantity CHECK (>= 0)`; assertions `quantity CHECK (> 0)`, `arrival_date < departure_date`, `UNIQUE (hotel_id, idempotency_key)`; movements typed `ASSERTION>0 / REVERSAL<0`, **append-only** via trigger; ≥1 live assertion per reservation per property | extract `:130-149` |
| Phase 2 idempotency | replay returns recorded result; key bound to request hash (`IDEMPOTENCY_CONFLICT`); claim+completion share one transaction | extract `:157-175` |
| Phase 2 operation identity | every mutating call carries `operationId` + `availabilityOperationKey`; reservation/assertion identity never substitutes | extract `:177-184` |
| Phase 2 lock ordering | fixed global order; capacity decided only from values read under the lock; Phase 3 prefix: journal claim → state row → sorted balance locks → `reservations` row | extract `:204-215` |
| Phase 2 outcomes | `ACTIVE / REJECTED / IDEMPOTENCY_CONFLICT / OPERATION_IN_PROGRESS / CURRENT_ASSERTION_MISSING / ASSERTION_LINK_MISMATCH / ASSERTION_DATE_NOT_ACTIVE / REPLACEMENT_PENDING` | extract `:217-231` |
| Phase 3 D-0 / D-1 | reservation quantity = candidate count (D-0); canonical status classification — `RESERVED` consumes, `CHECKED_OUT` holds without releasing, unclassifiable ⇒ `UNRESOLVED` (D-1) | extract §7; Phase 3 spec `:191` |
| Phase 3 D-15 | operation key derivation: HTTP `Idempotency-Key`; channel upstream identity; scheduled durable per-item identity | extract §7 (spec §22) |
| Phase 4 INV-1 / INV-8 / INV-9 | hotel isolation incl. raw SQL; **availability never silently clamps business facts** (invalid ⇒ flagged `UNRESOLVED` with provenance); overbooking explicit only | `13_…:593,600,601` |
| Phase 4 INV-12/13/14/15 | atomic counter/reservation changes; deterministic invalid-state handling (`CONFLICT`, no silent last-write-wins); core inequalities; no lost updates / version monotonicity | `13_…:604-607` |
| Phase 4 §3.2 duplicate-truth rules | exactly one producer of sellable availability; one consumption ledger; reservation is truth for whether consumption stands; pickup record is association truth; every rule hotel-scoped | `13_…:157-163` |
| Phase 4 §11 authority | A1 sole producer; A3 rebuilt on top of A1 with labeled interim view; eligibility filters single-placed; four fact kinds combined only by A1 | `13_…:429-463` |
| Phase 4 §19 reconciliation | no stored read model; stale-by-design prohibited; legacy-vs-new comparison read-only, **legacy never wins**; drift ⇒ flagged fact, never auto-rewritten | `13_…:737-759` |
| Phase 4 concurrency invariants | no lost updates; conflict is an error; exactly-once counter mutation per business event; version monotonicity (mechanism deferred) | `13_…:470-476` |

### 5.3 Preservation rule

Every rule in §5.1–§5.2 remains binding for Phase 5 unchanged. They may be **restated, referenced, or enforced** by Phase 5 requirements; they may **not** be amended except through the Stage 2 §5 amendment protocol with new evidence (G-11).

## 6. Phase 5 Stage 2 Decisions Incorporated

### 6.1 Decision items → this specification

| DS | Stage 2 status (closed) | Decision carried here | Sections |
|---|---|---|---|
| **DS-01** Restriction authority | `NEW DECISION` (Option A) + sub-items: 01.1/01.7 `PRESERVED`, 01.3/01.4 `NEW`, **01.5/01.6 `UNRESOLVED — EVIDENCE REQUIRED`** | Build the authoritative `RestrictionEvaluator` over the evidenced store set; conflict ⇒ `UNRESOLVED`; absence ⇒ no restriction; fail-closed unchanged; interim gate (flag OFF) | §13, §14, §29, §31 |
| **DS-02** Frontend read contract | `NEW DECISION` (Option C) | Two sanctioned contracts: `/snapshot` canonical, `/matrix` sanctioned projection; bed-type partition conserves the authority figure | §10, §11, §12, §17, §18 |
| **DS-03** Legacy rates/CRS disposition | `NEW DECISION` (retire/re-point per §10.2) | Endpoint-by-endpoint dispositions; no legacy signal gates or publishes availability | §19, §20, §21 |
| **DS-04** Restriction write ownership | `NEW DECISION` (Option A) + sub-item `UNRESOLVED` (CUTOFF) | Authority-owned restriction write path; A3 becomes a caller; CUTOFF store excluded pending E-4 | §23, §31 |
| **DS-05** Dual-write ordering | `NEW DECISION` (ordered removal) | Order: DS-01 operational → DS-03 dispositions → DS-05 leg removal; status quo preserved until gates | §24 |
| **DS-06** Occupancy / KPI | availability figures `RESOLVED`; analytics contract `PRESERVED` as `DEFERRED` | Availability-labelled figures from authority; occupancy = declared reporting | §17, §34 |
| **DS-07** Channel push scope | `PRESERVED` → `DEFERRED` | Out of scope; guard rule against pretend publication | §34, §33 |
| **DS-08** Multi-property scope | `PRESERVED` → `DEFERRED` | Out of scope; all rules single-hotel | §22, §34 |
| **DS-09** Test-execution policy | `NEW DECISION` | Executed-evidence duty per code-changing stage; config declaration duty | §36 (Stage 4 inputs) |
| **DS-10** Flag governance | `NEW DECISION` + rollout `PRESERVED` | One env `FEATURE_*` mechanism; no code flips; `gba.a3.authoritative` ON gate | §28 |
| **DS-11** UI repair / route collision / delete | part 1 `NEW`; part 2 `PRESERVED`+`NEW`; part 3 `UNRESOLVED` (U-2) | Canonical `/availability/logs` + `/availability/interval-update`; delete rejected when availability state exists; one route one owner | §21, §26, §27 |
| **DS-12** Physical/OOO/restriction inputs | ownership `RESOLVED`/`PRESERVED`; population `UNRESOLVED` (E-1) | Input ownership table; `room_inventory` legacy; input-validity rule | §13.3, §31 |

### 6.2 Adoption of the rule catalogue

All **51 business rules BR-5-001…BR-5-051** (Stage 2 §20) are adopted as binding provisions of this specification. §7 and §8 map each rule to its specification section and status; §29 states them as invariants; §30 states their acceptance conditions; §33 states their prohibited behaviors.

### 6.3 Non-reopening and Stage 3-requirement conversion

Stage 1/2 phrasing such as *"Stage 3 requirement"* or *"Stage 3 entry condition"* is discharged **by this document** (Phase 5 has no separate Requirements stage): each such requirement is converted here into a specification element with an acceptance condition. No requirement is postponed to a separate requirements document.

## 7. Stage 3 Requirements / Inputs Register

### 7.1 Register method

Every requirement-bearing construct in `02_BUSINESS_RULES_DECISIONS.md` is registered here with a stable ID (`S3R-nnn`) and exactly one status:

| Status | Meaning |
|---|---|
| **SPECIFIED** | Converted into an explicit requirement/contract/invariant/acceptance condition in this document. |
| **PRESERVED** | Content is a restatement of ratified Phase 1–4 authority; carried unchanged (§5). |
| **BLOCKED** | Incorporated as a complete specification whose finalization/operationalization cannot close until a named dependency (`DS-xx` / `E-x` / `BLK-P5-xx`) closes — recorded in §31, never defaulted. |
| **DEFERRED WITH EXPLICIT AUTHORITY** | Not specified because a ratified prior phase deferred it; authority cited. |

Nothing may be omitted: every class below is fully enumerated, and §8 maps each item to its specification section.

### 7.2 Input classes identified in Stage 2

| Class | ID range | Source in Stage 2 | Count |
|---|---|---|---|
| A — Business rules | S3R-001…051 (≡ BR-5-001…051) | §20 catalogue | 51 |
| B — Blocker requirements | S3R-052…057 (≡ BLK-P5-01…06) | §21 "Stage 3 requirement" column | 6 |
| C — Evidence requirements | S3R-058…065 (≡ E-1…E-8) | §30.1 | 8 |
| D — Stage-3 guardrails | S3R-066…077 (≡ G-1…G-12) | §30.2 | 12 |
| E — DS entry conditions | S3R-078…089 (≡ DS-01…DS-12) | §31.1 "Stage 3 entry condition" column | 12 |
| F — Finding-level requirements | S3R-090…116 (≡ F-01…F-27) | §23 "Stage 3 requirement" column | 27 |
| | | **Discrete items** | **116** |
| G — Cross-cutting duty statements (consolidations of classes A–F; members already counted) | S3R-G1…G6 | §8.3 interim rules, §9.2 bed-type decision, §10.2 endpoint dispositions, §19.1 input table, §22 ordering + §28 boundary, §29 carry-overs, §25/§26/§27 duty lists | 6 groups |

### 7.3 Register detail

#### 7.3.1 Class A — Business rules (S3R-001…051)

Fully enumerated in §8.2 with per-rule status and specification section. Summary: **37 SPECIFIED, 10 PRESERVED, 4 BLOCKED** (S3R-001/E-1, S3R-024/E-4, S3R-044/E-7, S3R-046/E-1), 0 deferred.

#### 7.3.2 Class B — Blocker requirements (S3R-052…057)

| ID | Requirement (verbatim intent from §21) | Status | Specification |
|---|---|---|---|
| S3R-052 (BLK-P5-01) | "Restriction evaluator over the evidenced store set with BR-5-002/003 semantics, proven by tests incl. conflict and absence cases, plus recorded U-1 runtime confirmation and U-3 population evidence" | **BLOCKED BY DS-01 / E-1 / E-3** | §13 (contract), §30 AC-08/AC-09, §31 |
| S3R-053 (BLK-P5-02) | "UI affordances implemented against the authority-owned restriction write contract; until then, affordances visibly disabled — never 404" | **BLOCKED BY DS-04** | §21.5, §17, §30 AC-45, §31 |
| S3R-054 (BLK-P5-03) | "Retire/re-point in the order DS-01 → DS-03 reads → DS-05 write leg; no step skipped" | **BLOCKED BY DS-01/DS-03** (ordering is itself specified) | §24, §30 AC-32/AC-25, §31 |
| S3R-055 (BLK-P5-04) | "Executed suite evidence recorded in each stage artifact" | SPECIFIED | §36, §30 AC-43 |
| S3R-056 (BLK-P5-05) | "Flag inventory documented; no code-level flips" | SPECIFIED | §28, §30 AC-37/AC-39 |
| S3R-057 (BLK-P5-06) | "Explicitly out of scope; may not be partially implemented under another heading" | DEFERRED WITH EXPLICIT AUTHORITY (P-19, P-20, TR-14.3) | §34, §33 G-9 |

#### 7.3.3 Class C — Evidence requirements (S3R-058…065)

| ID | Evidence required (§30.1) | Closes | Gates | Status |
|---|---|---|---|---|
| S3R-058 (E-1/U-3) | Population audit: per-hotel row counts, recency, writer identification for `room_inventory`, `out_of_order`, `out_of_service`, `rate_restrictions` + six A3 restriction tables | DS-12, DS-01.5 | BLK-P5-01 | **BLOCKED — E-1** (§31) |
| S3R-059 (E-2) | Precedence-intent evidence (only if actual conflicts exist) | DS-01.3 explicit ranking | nothing (fail-closed default stands) | DEFERRED WITH EXPLICIT AUTHORITY (conflict ⇒ `UNRESOLVED` is the decided default) |
| S3R-060 (E-3/U-1) | Executed confirmation that the production DI binding asserts/rejects as the static chain predicts | DS-01.6 | BLK-P5-01, BR-5-037 | **BLOCKED — E-3** (§31) |
| S3R-061 (E-4) | Business meaning of `restrictions` (`rate_code='CUTOFF'`) and the `zero_sell_value` domain | DS-04 sub-item, BR-5-024 | evaluator scope for that store only | **BLOCKED — E-4** (§31) |
| S3R-062 (E-5/U-5) | Inventory of out-of-repo consumers of `/rates/engine/*` | DS-03 retirement schedule | retirement **timing only** (disposition unchanged) | **BLOCKED — E-5 (schedule only)** (§31) |
| S3R-063 (E-6/U-6) | Live behavior of the OTA overbooking branch | DS-03 (BR-5-020) | none — rule already issued | SPECIFIED (evidence informative; §27) |
| S3R-064 (E-7/U-2) | Runtime winner of the `GET /tax-rates` collision | DS-11 part 3 | BR-5-044 execution | **BLOCKED — E-7** (§31) |
| S3R-065 (E-8/U-4) | Executed baseline reconciliation | DS-09 | first Stage 3+ executed exit | **BLOCKED — E-8** (§31) |

Evidence items are **evidence blockers, not implementation tasks** (brief §23): none of them converts into a coding task inside this document.

#### 7.3.4 Class D — Guardrails (S3R-066…077 ≡ G-1…G-12)

All **SPECIFIED** as binding prohibitions in §33 (each guardrail reproduced with its ID) and as invariants in §29. G-1 additionally constrains this stage itself (no schema/config/test changes made).

#### 7.3.5 Class E — DS entry conditions (S3R-078…089 ≡ DS-01…DS-12, Stage 2 §31.1)

| ID | Entry condition (Stage 2) | Status | Discharged in |
|---|---|---|---|
| S3R-078 (DS-01) | Evaluator requirement drafted; E-1 and E-3 scheduled | SPECIFIED (requirement drafted in §13; evidence scheduled §31) | §13, §31 |
| S3R-079 (DS-02) | Contract types + eligibility-surfacing requirements specified | SPECIFIED | §11, §12, §17, §30 AC-16…AC-22 |
| S3R-080 (DS-03) | E-5 external-consumer inventory before retirement schedule | SPECIFIED (dispositions fixed; schedule BLOCKED — E-5) | §20, §21, §31 |
| S3R-081 (DS-04) | Write-path requirement; E-4 before any CUTOFF migration | SPECIFIED | §23, §31 |
| S3R-082 (DS-05) | Ordering stated in requirements verbatim | SPECIFIED | §24 (ordering stated verbatim) |
| S3R-083 (DS-06) | Display rules specified; analytics contract not started | SPECIFIED (analytics contract DEFERRED P-20) | §17, §34 |
| S3R-084 (DS-07) | Out-of-scope declaration carried | DEFERRED WITH EXPLICIT AUTHORITY (P-19) | §34 |
| S3R-085 (DS-08) | Out-of-scope declaration carried | DEFERRED WITH EXPLICIT AUTHORITY (TR-14.3) | §22, §34 |
| S3R-086 (DS-09) | E-8 baseline reconciliation in first executed stage | SPECIFIED (duty carried to §36; E-8 blocked) | §36, §31 |
| S3R-087 (DS-10) | Config documentation task; no code flips | SPECIFIED | §28, §36 |
| S3R-088 (DS-11) | Canonical contracts fixed; E-7 before route-owner change | SPECIFIED (owner BLOCKED — E-7) | §21, §26, §31 |
| S3R-089 (DS-12) | E-1 population audit | SPECIFIED (ownership decided §13.3; population BLOCKED — E-1) | §13.3, §31 |

#### 7.3.6 Class F — Finding-level requirements (S3R-090…116 ≡ F-01…F-27, Stage 2 §23)

| ID | Finding requirement (condensed from §23) | Status | Specification |
|---|---|---|---|
| S3R-090 (F-01) | Evaluator requirement + conflict/absence cases + U-1/U-3 evidence | **BLOCKED — BLK-P5-01** | §13, §30 AC-08…AC-11, §31 |
| S3R-091 (F-02) | Snapshot/projection consumer requirements + consumption-count acceptance | SPECIFIED | §10, §11, §17, §36 |
| S3R-092 (F-03) | Authority-owned restriction read/write requirements; overlay removal | SPECIFIED | §23, §33 |
| S3R-093 (F-04) | Ordered removal requirement + drift-monitoring via reconciliation | SPECIFIED | §24, §25 |
| S3R-094 (F-05) | Eligibility consult via authority; absent-row default removed | SPECIFIED | §16.2, §20 |
| S3R-095 (F-06) | Route retirement (or de-availability + hotel scoping) requirement | SPECIFIED | §20, §21, §22 |
| S3R-096 (F-07) | Booking gate via authority eligibility; book path through reservation create | SPECIFIED | §16.1, §20, §21 |
| S3R-097 (F-08) | Canonical route/payload requirement; wiring gated by DS-04 | **BLOCKED — DS-04 (BLK-P5-02)** for wiring; contract itself specified | §21.5, §17, AC-26, AC-45 |
| S3R-098 (F-09) | Remove/re-point each client formula; conservation law for bed-type split | SPECIFIED | §12.4, §18, §33 |
| S3R-099 (F-10) | Cell mapping from projection + eligibility surfacing (quick-book) | SPECIFIED | §17, §30 AC-22 |
| S3R-100 (F-11) | Availability figures re-pointed to authority; `room_inventory` treated as legacy | SPECIFIED | §20, §13.3 |
| S3R-101 (F-12) | Single-source acceptance: no two screens disagree on available | SPECIFIED | §30 AC-01 |
| S3R-102 (F-13) | Shared contract type requirement | SPECIFIED | §11.6, §17 |
| S3R-103 (F-14) | Document "no cache"; future cache must implement TR-10.3 | SPECIFIED | §28.4, §11.5 |
| S3R-104 (F-15) | Out of scope; non-authoritative log marked as such | DEFERRED WITH EXPLICIT AUTHORITY (P-19) + SPECIFIED guard | §34, §33 |
| S3R-105 (F-16) | Deterministic identity requirement (preserved Phase 2 §5.4) | PRESERVED + SPECIFIED | §5.2, §24.5, AC-33 |
| S3R-106 (F-17) | Env declaration + zero-harness-skip reporting | SPECIFIED | §36, AC-43 |
| S3R-107 (F-18) | Single flag mechanism + config documentation | SPECIFIED | §28 |
| S3R-108 (F-19) | Route-property authorization requirement; guard divergence reconciled | SPECIFIED | §22, AC-13 |
| S3R-109 (F-20) | Runtime winner verification + single owner | **BLOCKED — E-7** | §21.7, §31 |
| S3R-110 (F-21) | Removal gated behind behavior-preserving migration | SPECIFIED (gate) | §33 G-12, §36 |
| S3R-111 (F-22) | Baseline reconciliation in first executed stage | SPECIFIED (E-8 at exit) | §36, §31 |
| S3R-112 (F-23) | Correct drift in Stage 4 doc chores; never cite stale lines as live | SPECIFIED | §36 |
| S3R-113 (F-24) | Phase 4 actions remain owned by Phase 4; Phase 5 may not assume them done | SPECIFIED | §32 |
| S3R-114 (F-25) | Consolidation behind the sanctioned read surfaces | SPECIFIED | §36 (Stage 4 hygiene, gated) |
| S3R-115 (F-26) | Parameterized reads/writes on the authority path (write side here) | SPECIFIED | §23, §33 |
| S3R-116 (F-27) | Deterministic error-contract detection; U-6 live check | SPECIFIED (E-6 informative) | §27, AC-27 |

#### 7.3.7 Class G — Cross-cutting duty statements (consolidations; members already counted)

| ID | Stage 2 source | Nature | Specification |
|---|---|---|---|
| S3R-G1 | §8.3 interim rules (4) | Binding now: flag OFF; structural consumption with eligibility surfacing; no write-gating re-point; stub kept as rollback artifact | §13.6, §28.3, §24 |
| S3R-G2 | §9.2 bed-type grouping decision | Display-only partition with conservation law + deterministic residue | §12.4 |
| S3R-G3 | §10.2 endpoint dispositions (8 rows) | Normative API dispositions (retire/re-point/retain) | §20, §21.4 |
| S3R-G4 | §19.1 input-ownership table (11 stores) | Input ownership + applicability | §13.3 |
| S3R-G5 | §22 hard ordering + §28 migration-boundary constraint | B-1 (authority enablement) before B-2…B-6; B-4 highest risk | §24.2 |
| S3R-G6 | §25 / §26 / §27 duty lists (isolation 5, frontend 8, API 8) | Consolidated duty statements | §22, §17, §21 |

### 7.4 Status summary (nothing dropped)

| Status | Class A | B | C | D | E | F | Total |
|---|---|---|---|---|---|---|---|
| SPECIFIED | 37 | 2 | 1 | 12 | 12 | 24 | **88** |
| PRESERVED | 10 | 0 | 0 | 0 | 0 | 0 | **10** |
| BLOCKED (named dependency) | 4 | 3 | 6 | 0 | 0 | 3 | **16** |
| DEFERRED WITH EXPLICIT AUTHORITY | 0 | 1 | 1 | 0 | 0 | 0 | **2** |
| **Total** | 51 | 6 | 8 | 12 | 12 | 27 | **116** |

Plus 6 cross-cutting duty groups (S3R-G1…G6), all incorporated. **Silently dropped: 0.**

## 8. Requirements-to-Specification Traceability

### 8.1 Method

`Stage 2 requirement/input (S3R) → this specification section → domain rule/contract → acceptance condition (AC in §30)`. Blocked items additionally name their dependency (§31).

### 8.2 Class A — every business rule mapped (BR-5-001…051)

| ID (S3R) | Rule (short) | Section(s) | Status |
|---|---|---|---|
| 001 | Evaluator scope over evidenced store set | §13.3 | **BLOCKED — E-1** (scope confirmation) |
| 002 | Absence semantics: no row ⇒ no restriction, provenance recorded | §13.4 | SPECIFIED (AC-09) |
| 003 | Conflict ⇒ dimension `UNRESOLVED` + `sourceConflicts`; no precedence | §13.4 | SPECIFIED (AC-08); explicit ranking DEFERRED (E-2) |
| 004 | Outcome `RESOLVED` iff every applicable dimension resolved | §13.5 | SPECIFIED (AC-10) |
| 005 | Fail-closed propagation (restated) | §14 | PRESERVED (P-3/P-5/P-7) |
| 006 | Blocked semantics + `sellLimit` tightening (restated) | §13.5, §15 | PRESERVED (P-3/P-4) |
| 007 | Allotment stop sale (restated) | §13.5 | PRESERVED (P-14) |
| 008 | Single combination — only A1 combines (restated) | §9.4, §33 | PRESERVED (P-11/P-12/INV-18) |
| 009 | Interim operational gate (flag OFF, no write-gating cutover, stub retained) | §13.6, §28.3 | SPECIFIED |
| 010 | Two sanctioned read contracts; no third contract | §10, §21.1 | SPECIFIED (P-17/P-18 embedded) |
| 011 | Eligibility surfacing; unresolved never shown as certainty | §17.3, §15 | SPECIFIED (AC-19) |
| 012 | No client-side availability math; projection conservation | §12.4, §18, §33 | SPECIFIED (AC-18/AC-20) |
| 013 | One shared contract type (snapshot + projection) | §11.6, §17.2 | SPECIFIED |
| 014 | `/reconciliation` is the sanctioned comparison, read-only | §25 | PRESERVED (P-1/P-16) |
| 015 | `GET /rates/availability` retired/de-availability'd, scoped | §20, §21.4 | SPECIFIED (AC-23) |
| 016 | `/rates/engine` reads re-pointed; pricing retained | §20, §21.4 | SPECIFIED |
| 017 | `/rates/engine/book` retired as booking path | §20, §21.4 | SPECIFIED (execution gated BLK-P5-03) |
| 018 | Legacy counters: no new readers; writer ceases; table read-only | §20, §24.4 | SPECIFIED; retirement schedule BLOCKED — E-5 |
| 019 | FO upgrade gate consults authority; no absent-row default | §16.2, §20 | SPECIFIED |
| 020 | Deterministic error contract; no message-substring detection | §27 | SPECIFIED (AC-27) |
| 021 | Authority-owned restriction write path; raw writes prohibited post-cutover | §23 | SPECIFIED (AC-30) |
| 022 | Restriction reads for display from authority projection | §23, §12.3 | SPECIFIED |
| 023 | No second combination (A3 overlay removed) | §23, §33 | SPECIFIED (AC-02) |
| 024 | Unproven stores not promoted (CUTOFF) | §23.5 | **BLOCKED — E-4** |
| 025 | Single write truth on modify after ordering gates | §24.4 | SPECIFIED (AC-32) |
| 026 | No premature cuts — status quo preserved until gates | §24.3 | SPECIFIED (AC-32) |
| 027 | Deterministic write identity (restated) | §5.2, §24.5 | PRESERVED (Phase 2 §5.4) |
| 028 | Availability-labelled figures come from authority | §17.4, §33 | SPECIFIED (AC-42) |
| 029 | Occupancy KPIs = declared reporting | §17.4 | SPECIFIED (AC-42) |
| 030 | No outbound publication from non-authority source | §34, §33 | SPECIFIED (guard); push contract DEFERRED (P-19) |
| 031 | Every rule/read/write hotel-scoped; no cross-hotel aggregation | §22.2 | PRESERVED (TR-14.3/INV-1) |
| 032 | `hotelId === 'default'`/absent context rejected at authority entry | §22.3 | SPECIFIED (AC-13) |
| 033 | Executed evidence per code-changing stage | §36 | SPECIFIED (AC-43) |
| 034 | `AVAILABILITY_TEST_DATABASE_URL` declared in config docs/CI | §36 | SPECIFIED (Stage 4 duty) |
| 035 | Single flag mechanism (env `FEATURE_*`); flag changes operational | §28.2 | SPECIFIED (AC-37) |
| 036 | All seven flags declared in `compose.yaml`/`.env.example` | §28.2, §36 | SPECIFIED (Stage 4 duty) |
| 037 | `gba.a3.authoritative` ON gate (evaluator + E-1 + E-3 + soak) | §28.3 | SPECIFIED; satisfaction BLOCKED — E-1/E-3 |
| 038 | Preserved rollout sequence (cascade ON at deploy; wash blocked; canonicalRead→soak→canonicalWrite) | §28.3 | PRESERVED (P-21) |
| 039 | Freshness honesty: `LIVE_READ`, no cache today, future cache real | §11.5, §28.4 | SPECIFIED |
| 040 | Canonical API names `/availability/logs`, `/availability/interval-update` | §21.5 | SPECIFIED (AC-26) |
| 041 | No silent 404 affordance | §17.5, §21.5 | SPECIFIED (AC-45) |
| 042 | Delete never releases (terminal-only; no availability op) | §26.4 | PRESERVED (Decision 12/P-8/P-9) |
| 043 | Delete with availability state ⇒ deterministic business rejection | §26.4, §27 | SPECIFIED (AC-34) |
| 044 | One route, one owner | §21.7 | **BLOCKED — E-7** (owner selection) |
| 045 | `room_inventory` legacy — no availability figure solely from it | §20, §13.3 | SPECIFIED (INV-P5-31) |
| 046 | Input validity: recorded writer/population or flagged provenance | §13.3, §29 | SPECIFIED; population recording BLOCKED — E-1 |
| 047 | Route-property authority (authorization property = data property) | §22.4 | SPECIFIED (AC-13) |
| 048 | No guard opt-out for availability surfaces | §22.5 | SPECIFIED |
| 049 | Raw SQL carries `hotel_id` | §22.6 | PRESERVED (TR-14.1) |
| 050 | No legacy availability signal gates or publishes | §20, §33 | SPECIFIED (AC-24/AC-25) |
| 051 | Unscoped reads not preserved for compatibility | §22.7 | SPECIFIED |

### 8.3 Classes B–F mapped

| Class | Items | Map to |
|---|---|---|
| B — blocker requirements (6) | §7.3.2 | §13, §21.5, §24, §36, §28, §34 + AC-08/09/12, AC-45, AC-32/25, AC-43, AC-37/39 |
| C — evidence requirements (8) | §7.3.3 | §31 (affected sections named per item), AC-07…AC-12 gate |
| D — guardrails (12) | §7.3.4 | §33 (prohibitions), §29 (invariants) |
| E — entry conditions (12) | §7.3.5 | §13, §11, §12, §17, §20, §21, §23, §24, §28, §31, §36 |
| F — finding requirements (27) | §7.3.6 | full chain in §35.1 |

### 8.4 Completeness statement

All 116 discrete inputs + 6 duty groups are accounted for with a section and (where applicable) an acceptance condition. **Requirement state distribution: 88 SPECIFIED · 10 PRESERVED · 16 BLOCKED (named dependency) · 2 DEFERRED WITH EXPLICIT AUTHORITY · 0 dropped.**

## 9. Final Availability Authority Model

### 9.1 Canonical availability authority

**The Availability context (A1) is the single canonical authority for availability.** It consists of:

| Component | Domain role |
|---|---|
| Source adapter | Reads the four ratified fact kinds hotel-scoped: **physical quantity** (`rooms` + status exclusions + `out_of_order`/`out_of_service`), **selling permission** (restriction stores + allotment stop sale), **reservation commitment** (reservation consumption), **history/GBA** (block allocations, allotment quotas) — TR-1.1 (P-11) |
| Snapshot calculator | The only combination engine: §11.3 arithmetic (P-4), producing `physicalAvailable` / `sellableAvailable` / consumption figures |
| Restriction evaluator | Combines selling-permission stores into per-dimension outcomes (§13) — the DS-01 deliverable |
| Assertion engine (Phase 2/3) | The only write-side authority: balances, movements, journal, fail-closed rejection (§5.2) |

Exactly **one** producer of sellable availability exists (P-12, INV-18, `10_…:114`). No component outside A1 may combine selling permission with quantity (BR-5-008).

### 9.2 Canonical read model

Consumers needing availability truth read:

1. **`GET /properties/:propertyId/availability/snapshot`** — the canonical domain read (§11).
2. **`GET /availability/matrix`** — the sanctioned **projection** for grid surfaces (§12), whose values are authority-sourced when `gba.a3.authoritative` is ON and labeled as a derived view while OFF.
3. **`GET /properties/:propertyId/availability/reconciliation`** — the sanctioned read-only legacy-vs-authority comparison (§25).

No third contract may publish availability (BR-5-010).

### 9.3 Projections

A projection is a **shape-preserving presentation of authority facts**, never a second computation. Current sanctioned projection: the A3 availability matrix (response shape preserved per P-18/T-38). A projection:

- sources every availability/eligibility value from A1 (§12.2),
- may add fields (e.g. eligibility) but may not remove the preserved shape,
- may partition a room-type figure across bed types only under the conservation law (§12.4),
- carries the interim `label: 'derived view, not sellable availability'` while not authority-backed (P-17),
- never becomes an independent read for eligibility decisions.

### 9.4 Consumer responsibility

Consumers MAY: read sanctioned contracts; perform **presentation-only transforms** (grouping, formatting, ordering, display partitioning under conservation); surface eligibility and provenance; match deterministic error codes.

Consumers MAY NOT: compute available/sellable/occupancy-labelled figures (BR-5-012); recreate restriction logic (BR-5-008); treat projections as a second authority; treat `UNKNOWN` as `ELIGIBLE` (P-3); publish availability from non-authority sources (BR-5-030); aggregate across hotels (BR-5-031).

### 9.5 Prohibited authority (may never be treated as availability truth)

| Source | Why prohibited |
|---|---|
| Legacy `availability` counters + `inventory.domain-service` math | Legacy retained read-only, never authoritative (P-16); drift-by-design (F-04) |
| `GET /rates/availability` ad-hoc KPI | unscoped, div-by-zero prone, not an availability computation (F-06); retired (BR-5-015) |
| `/rates/engine/{availability,restrictions,book}` signals | legacy counter engine (F-07); retired/re-pointed (BR-5-016/017/050) |
| `quote.available` / `blockReasons` as gating truth | legacy signal; re-pointed to authority eligibility (BR-5-016/050) |
| A3 raw-SQL computations (`max(0, physical − ooo − reserved − …)`) | second engine (F-03); removed or projection-only (BR-5-008/023) |
| A3 `available: hasZeroSell ? 0 : authority.available` overlay | second combination site (F-03); prohibited (BR-5-008/023) |
| `room_inventory`-sourced `availableRooms` | no writer; legacy/derived only (F-11, BR-5-045) |
| Client-side formulas (audit §5.3, ≥12) | independent computation (F-09); prohibited (BR-5-012) |
| `channel_availability_log` caller-supplied numbers | non-authoritative; feeds nothing (BR-5-030) |
| Mock/hardcoded figures (`78.4%`, `total_rooms \|\| 200`, admin mocks) | may not represent live truth (F-09/F-12, BR-5-029) |
| Quick-book literal `available: 0` | broken display (F-10); must map projection + eligibility (BR-5-011/012) |
| FO `inventoryDomain.isAvailable` absent-row `true` | legacy gate with fail-open default (F-05); never gates a write (BR-5-019/050) |

### 9.6 Canonical truth vs presentation (explicit distinction)

| Property | Canonical business truth | Presentation / projection / derived |
|---|---|---|
| Produced by | A1 only (snapshot + assertion) | sanctioned projection or frontend display layer |
| Contains | quantities, consumption, restriction outcomes, eligibility, provenance, unresolved sources | numbers copied from truth + labels + partitions |
| May drive bookings/assertions | yes (through the assertion port only) | **never** |
| May be recomputed by consumers | no | yes — display transforms only |
| Failure mode when inputs unknown | `UNRESOLVED`/`UNKNOWN` declared (P-2) | must surface unresolved state (BR-5-011), never invent certainty |

## 10. Canonical Read Contract

**REQ-10.1** There are exactly two sanctioned availability reads for consumers (plus reconciliation, §25): the **snapshot** (canonical domain read) and the **matrix** (sanctioned projection). Any surface needing provenance, eligibility, restriction outcomes, or unresolved detail reads the snapshot; any grid/table surface reads the matrix projection.

**REQ-10.2 (consumer migration duty)** Availability-bearing consumers (availability page, rates tab, quick-book, dashboards, admin/mobile greenfield) must consume one of the sanctioned contracts; consumers that cannot yet migrate must satisfy BR-5-041 (wired or explicitly hidden with declared status) and must not present legacy numbers as authority (BR-5-011 label duty). Consumption acceptance for F-02: the authority read surface must gain consumers as migration proceeds; "zero consumers" (audit §15.3-E1) is the measured starting state, not an allowed end state.

**REQ-10.3** The frontend obtains availability exclusively through the shared contract types (§11.6) — no hand-rolled payload duplicates (BR-5-013).

**REQ-10.4** No new availability endpoint may be invented for convenience (plan §8 discipline `14_…:813-840`). Adding a sanctioned contract requires a decision amendment (§3.4).

## 11. Snapshot Domain Contract

### 11.1 Endpoint and access

`GET /properties/:propertyId/availability/snapshot?roomType&arrivalDate&departureDate&channelCode&rateCode` — **read-only** (P-1). Authorization: actor required (403 without actor), property membership + permission codes (`availability:read`, `reservations:read`, `reservations:occupancy:read`, `*`, `platform:admin:full`), and `hotelId === 'default'` rejected (§22.3). Writes never occur through this contract; mutations go through the Phase 2 assertion port only (P-1).

### 11.2 Request semantics

| Element | Semantics |
|---|---|
| `propertyId` (route) | the property context; authoritative for both authorization and data access (BR-5-047) |
| `roomType` | single room type per request; response is per-date rows for that room type |
| `arrivalDate`/`departureDate` | stay range, **half-open `[arrivalDate, departureDate)`**; one response row per date in range; departure date excluded (aligned with assertion `arrival < departure`, §5.2) |
| `channelCode`/`rateCode` | optional qualifiers for restriction/eligibility resolution; when absent, evaluation uses the default (hotel+roomType+date) scope |
| Missing/invalid property context | rejected deterministically (§27.1/27.2) — never defaulted |

### 11.3 Response semantics (quantity and calculation)

Per-date payload (verified field set, audit §7.1): `physicalCount`, `oooCount`, `oosCount`, `unavailablePhysicalCount`, `reservationConsumption` (+ outcome), `gbaHeld/Picked/Released/Remaining`, `allotmentQuota/Picked/Released/Remaining`, `sellLimit`, `overbookingAllowance/Used`, `consumption`, `physicalAvailable`, `sellableCapacity`, `sellableAvailable`, `restrictionOutcome`, `unresolvedSources`, `bookingEligibility`, `sourceReferences`, `calculatedAt`, `freshness`.

Normative arithmetic (preserved, §5.2):

```
physicalCapacity        = max(0, physicalCount − unavailablePhysicalCount)
capacityWithOverbooking = physicalCapacity + max(0, overbookingAllowance)
sellableCapacity        = sellLimit === null ? capacityWithOverbooking
                                         : min(capacityWithOverbooking, max(0, sellLimit))
physicalAvailable       = max(0, physicalCapacity − consumption)
sellableAvailable       = max(0, sellableCapacity − consumption)
overbookingUsed         = max(0, consumption − physicalCapacity)
```

**Quantity semantics:** all quantities are non-negative room counts (integer domain); no path yields a negative figure; `sellLimit` can only tighten; overage is measured (`overbookingUsed`), not forbidden; consumption figures are `null` when the corresponding source is unresolved (never guessed).

### 11.4 Status, restriction, eligibility, unresolved fields

- `restrictionOutcome.status ∈ {RESOLVED, UNRESOLVED}` with `reasons`/dimension detail (§13).
- `bookingEligibility ∈ {ELIGIBLE, UNKNOWN, BLOCKED}` — semantics and duties in §14/§15.
- `unresolvedSources[]` — deduplicated source keys that could not be resolved; must be surfaced, never suppressed (G-2).
- `sourceReferences[]` — provenance (TR-10.4).

### 11.5 Freshness / stale-data semantics

`freshness: LIVE_READ` — values are computed on read; **no availability cache exists today** (BR-5-039). Stale-by-design is prohibited (TR-10.3): any future cache must implement real invalidation; the current `cache.delPattern('availability:…')` no-op may not be presented as a freshness mechanism (F-14). `calculatedAt` reports compute time.

### 11.6 Shared contract types

Exactly one shared TypeScript type for the snapshot payload and one for the matrix projection, living in `@xylo/shared` (or designated shared package), consumed by backend and frontend (BR-5-013, F-13). Consumers may not re-declare payload shapes (REQ-10.3). Type change = contract change = §3.4 discipline.

## 12. Matrix Projection Contract

### 12.1 Role and shape

`GET /availability/matrix` remains the wire contract for grid surfaces with **response shape preserved** (P-18, T-38). It is a projection, not a domain authority (§9.3).

### 12.2 Value sourcing

| Flag state | Availability values | Constraint |
|---|---|---|
| `gba.a3.authoritative` **OFF** (current, mandated interim per BR-5-009/037) | authority-derived values are not substituted; cells carry `label: 'derived view, not sellable availability'` | label must remain visible (BR-5-011); values may not be presented as sellable availability (P-17) |
| `gba.a3.authoritative` **ON** (gated: §28.3) | every availability/eligibility value sourced from the snapshot fact for the same inputs | no local arithmetic; no restriction overlay (§12.3) |

Flag-ON precondition: evaluator shipped + E-1 + E-3 + parity soak (BR-5-037) — currently **blocked** (§31). While the evaluator is unbound, flag-ON would zero every cell (`availability-sales.controller.ts:540`) and is therefore prohibited operationally (BR-5-009).

### 12.3 Restriction display

Restriction values shown in/on the projection come from the **authority projection** (BR-5-022); raw restriction reads for display (`:384-405`) are removed when DS-04's read cutover lands. Until then they are legacy reads that may not be combined with availability numbers (BR-5-008).

### 12.4 Bed-type partition (conservation law)

Matrix cells are keyed `roomType|bedType`. A bed-type split is a **display partition**, permitted only if it **conserves** the authority room-type figure:

- For each date: `Σ(bed-type cells for a room type) = authority figure for that room type` exactly.
- Rounding residue is assigned **deterministically** and **disclosed in the projection** — never dropped (BR-5-012).
- Bed-type cells are not independent availability numbers; they may not be summed, gated, or published outside this display purpose.

Rejected alternative (recorded): dropping bed-type granularity (breaks existing grid/quick-book layout without business justification).

### 12.5 Additive fields

The projection may add eligibility/state fields (`bookingEligibility`, unresolved markers) so consumers can meet the P-3 duty without a second call; additive fields do not change the preserved shape (BR-5-010).

## 13. Restriction Evaluator Domain Contract (DS-01)

### 13.1 What restriction evaluation means

Restriction evaluation is A1's determination of **selling permission** for `(hotel, roomType, stayDate [, rateCode, channelCode])`: for each restriction dimension, resolve the value from its store(s), declare the dimension `RESOLVED` (with value or explicit absence), `NOT_APPLICABLE`, or `UNRESOLVED` (with reason + provenance), then aggregate into a `RestrictionOutcome`. It is one of the four fact kinds whose combination into sellable availability is single-sourced in A1 (P-11) and is the missing implementation of TR-1.1's selling-permission kind (Stage 2 §7 Q5: the permanent-`UNRESOLVED` binding is an **implementation gap**, not a ratified rule).

**Dimension vocabulary (canonical mapping):**

| BR-5-001 label | Phase 1 contract dimension | Effect when true/out-of-range |
|---|---|---|
| `stopSell` | `closedToSell` | blocks all arrival-date selling for scope |
| `cta` | `closedToArrival` | blocks arrival on that date |
| `ctd` | `closedToDeparture` | blocks departure on that date |
| `minLos` / `maxLos` | `minLos` / `maxLos` | stay length outside range ⇒ blocked |
| `sellLimit` | `sellLimit` | caps capacity (tighten-only, §11.3) |
| `allotmentStopSale` | (allotment stop sale, TR-7.x) | selling permission for allotment intake; display remaining 0 |
| `channelRestriction` | (only once a writer exists) | channel-scoped selling permission |

### 13.2 Input inventory (scope set — BR-5-001)

| Store | In evaluator scope? | Basis |
|---|---|---|
| `close_to_arrival`, `close_to_departure`, `zero_sell_limits`, `room_type_sell_limits`, `minimum_length_of_stay`, `maximum_length_of_stay` | **YES** | in-repo writer (A3 `:676-707`, `:825-885`); Stage 2 §7 Q6 |
| `rate_restrictions` | **YES** | read by booking gate + adapter (`crs-engine.service.ts:149-163`, `unresolved-restriction.adapter.ts:17-35`) |
| `allotment_daily_quotas.stop_sale_active`, `allotment_stop_sales` | **YES** (already resolved by source adapter) | ratified TR-7.x (P-14); `availability-source.adapter.ts:65,129-145` |
| `channel_restrictions` | only **once a writer exists** | no writer/reader today (`unresolved-restriction.adapter.ts:44`) |
| `restrictions` (`rate_code='CUTOFF'`) | **NO — excluded pending E-4** | written by A3, read by nothing; meaning undefined (BR-5-024) |
| `room_inventory` | **NO** (not a restriction/authority input at all) | BR-5-045 (§13.3) |

Store scope confirmation is **BLOCKED BY E-1** (population evidence): a store remains in scope only while a writer/population source is identified (BR-5-046).

### 13.3 Physical / OOO / restriction input ownership (DS-12, evidence-based)

| Store | Role | Ownership decision |
|---|---|---|
| `rooms` (+ status exclusions `NON_INVENTORY_*`, `NON_PHYSICAL_*`, `isPhysicalRoomStatus`) | **the** physical inventory source | PRESERVED — authority definition stands (`availability-source.adapter.ts:40-49,60,71`) |
| `out_of_order`, `out_of_service` rows | physical-capacity reduction input, **read-only to Availability** | owned by room-state operations (Front Office/housekeeping); consumed read-only; **no in-repo writer identified (E-1)** |
| `rooms.room_status = 'OUT_OF_ORDER'` | second OOO channel inside `physicalCount` | PRESERVED |
| `room_inventory` | **not an authority input** | legacy/derived projection only; A3 `availableRooms` from it may not be presented as availability (BR-5-045) |
| six A3 restriction tables | restriction inputs | in evaluator scope; writer today is A3; ownership moves to the authority write path (DS-04, §23) |
| `rate_restrictions` | restriction input | in scope; writer unknown (E-1) |
| `allotment_daily_quotas.stop_sale_active`, `allotment_stop_sales` | ratified selling permission | PRESERVED (TR-7.x), already resolved by A1 |
| `restrictions` (`CUTOFF`), `channel_restrictions` | — | excluded (E-4 / no-writer) — §13.2 |

**Input-validity rule (BR-5-046):** each authority input store must have an identified writer/population source recorded; where none is found, its contribution is flagged with provenance (TR-10.4) or excluded — **never silently trusted** (no "absence = truth" for unproven stores).

### 13.4 Evaluation semantics

| Case | Rule | Requirement |
|---|---|---|
| Store has **no row** for the scope | that store imposes nothing; dimension `RESOLVED` with value "no restriction"; absence check recorded in `sources[]` (BR-5-002). Two independent legacy implementations agree (`crs-engine.service.ts:163`; A3 set membership `:430-447`) | AC-09 |
| Store **unreadable/failed** | that dimension `UNRESOLVED` with reason (BR-5-001 error behavior) | AC-07 |
| Two+ applicable stores **disagree** | dimension `UNRESOLVED`; `sourceConflicts[]` populated with store+value pairs; provenance retained; **no store silently prioritized; no precedence ranking invented** (BR-5-003; explicit ranking = E-2, §31) | AC-08 |
| Store unproven (no recorded writer/population) | contribution flagged with provenance or excluded (BR-5-046); absence may not be read as permission for that store | AC-40 |

### 13.5 Outcome semantics (preserved formulas)

- **Outcome status:** `RESOLVED` **iff** every applicable dimension is `RESOLVED` or `NOT_APPLICABLE`; any `unresolvedSources[]` entry ⇒ `UNRESOLVED` (BR-5-004). AC-10.
- **Blocked:** `closedToSell|closedToArrival|closedToDeparture = true`, or stay length outside `minLos`/`maxLos` ⇒ `bookingEligibility = 'BLOCKED'`, `sellableAvailable = 0` (BR-5-006, extract `:93-95`). AC-05.
- **sellLimit:** `sellableCapacity = min(capacityWithOverbooking, max(0, sellLimit))` — tighten-only (BR-5-006, P-4).
- **Allotment stop sale:** applied ⇒ selling display remaining 0; quantity/counters untouched; lift restores display; evaluation order stop-sale **before** remaining; per-quota conflict (daily flag vs applied records) ⇒ `UNRESOLVED` (BR-5-007, P-14/INV-17). AC-11 (stop sale) with conflict semantics per AC-08/AC-10.

### 13.6 Effect on sellability, eligibility, booking, and display

| Input state | `sellableAvailable` | `bookingEligibility` | Assertion/booking | Frontend duty |
|---|---|---|---|---|
| All applicable dimensions resolved, none blocking | computed value (may be genuine `0`) | `ELIGIBLE` | assert allowed; capacity checked ⇒ `INSUFFICIENT_CAPACITY` if 0 | show quantity (0 shown as certain zero, §15) |
| Resolved and blocking (§13.5) | `0` | `BLOCKED` | rejected (`CAPACITY_BLOCKED` semantics) | show distinct restricted/blocked state |
| Any source or dimension `UNRESOLVED` | `0` (**conservative floor**) | `UNKNOWN` | assertion persisted `REJECTED`/`UNRESOLVED_CAPACITY` ⇒ `AVAILABILITY_ASSERTION_REJECTED` (409), transaction rolled back | show unresolved/unknown — **never** a definite number |

**Booking/assertion duty:** assertion is permitted only when the snapshot for the affected dates resolves (`bookingEligibility` not `UNKNOWN`); fail-closed is the default, not an error path (P-5). **Frontend duty:** eligibility must be surfaced (BR-5-011); zeros derived from unresolved outcomes are never rendered as certainty.

### 13.7 Interim rules binding now (S3R-G1)

1. `gba.a3.authoritative` **must remain OFF** (BR-5-009/037).
2. Surfaces may consume the read contract structurally (hooks, types, wiring), but any rendered number applies BR-5-011.
3. **No consumer may be re-pointed onto the authority for write gating** until the evaluator exists and E-1/E-3 close (ordering behind DS-03/DS-05 — §24).
4. The unresolved stub binding remains available as a **rollback artifact**; removing it is not required by this contract.

### 13.8 Portions blocked pending evidence (preserved exact blockers)

| Portion | Blocked by | Not decided here |
|---|---|---|
| Final evaluator store list (per-store inclusion after audit) | **E-1** (DS-01.5) | population facts |
| Operational declaration that the chain asserts as predicted | **E-3** (DS-01.6) | runtime observation |
| Explicit precedence ranking when stores disagree | **E-2** (DS-01.3) | product/operational intent; default remains conflict ⇒ `UNRESOLVED` |
| Inclusion/meaning of `restrictions` (`rate_code='CUTOFF'`) and `zero_sell_value` domain | **E-4** (DS-04 sub-item) | business meaning |

## 14. UNRESOLVED Capacity Semantics

**REQ-14.1 (meaning).** `UNRESOLVED` / `bookingEligibility: 'UNKNOWN'` is a **statement about knowledge**, not a capacity value: a source or dimension declined to assert what it cannot compute (P-2). It carries `reasons` + `unresolvedSources` + provenance.

**REQ-14.2 (conservative floor, not zero-with-certainty).** Under unresolved inputs the response fields `physicalAvailable = 0` and `sellableAvailable = 0` are the **conservative floor** mandated by P-3. They are *not* a business fact of "no rooms". Reporting or rendering them as a definite count is a contract violation (BR-5-011).

**REQ-14.3 (duties).** A caller may not treat `UNKNOWN` as `ELIGIBLE` (P-3); must not suppress `unresolvedSources` (G-2); must not bypass `unassertableReason` (G-2); an assertion against unresolved capacity persists `REJECTED` with `UNRESOLVED_CAPACITY` and surfaces `AVAILABILITY_ASSERTION_REJECTED` (409) with nothing committed (P-5, P-6).

**REQ-14.4 (never assume).** "When capacity cannot be safely resolved, nothing commits" (P-5); "never assume zero or available" (P-7). Fail-open (option C of Stage 2 §8) is permanently rejected (§33 G-2).

**REQ-14.5 (distinctions preserved).** UNRESOLVED capacity is distinct from: **actual zero** (all sources resolved, quantity genuinely 0), **blocked** (resolved restriction refuses selling), and **system error** (the request itself failed). See §15.

## 15. Availability State Model

### 15.1 State space (never collapsed for display convenience)

| State | Meaning | Source | Allowed consumer behavior | Prohibited interpretation | Frontend representation requirement | Booking/assertion implication |
|---|---|---|---|---|---|---|
| **A. ELIGIBLE with quantity Q > 0** | knowledge complete; no blocking restriction; Q rooms sellable | A1 snapshot | display Q; attempt booking | — | numeric availability from sanctioned contract | assertion subject to capacity check under lock |
| **B. ELIGIBLE with quantity Q = 0 (actual zero)** | knowledge complete; genuinely no sellable rooms (restriction not blocking) | A1 snapshot | display 0 **with certainty** | presenting 0 caused by unresolved inputs as this state | numeric 0, labeled available | `INSUFFICIENT_CAPACITY` on assert |
| **C. BLOCKED** | resolved restriction refuses selling (stop-sale/closed/LOS) | `restrictionOutcome` + BR-5-006/007 | display restricted/closed state | rendering BLOCKED as UNKNOWN or vice versa; treating as failure | distinct blocked state (BR-5-011) | rejected (`CAPACITY_BLOCKED` semantics); quantity untouched by allotment stop sale |
| **D. UNKNOWN / UNRESOLVED capacity** | at least one source/dimension unresolved | `unresolvedSources`, `restrictionOutcome.status='UNRESOLVED'` | display "unresolved/unknown"; pass provenance through | rendering the floor 0 as certainty; silently converting to 0-as-fact; suppressing reasons | distinct unknown state; never a bare number | assertion `REJECTED`/`UNRESOLVED_CAPACITY` ⇒ `AVAILABILITY_ASSERTION_REJECTED` (409) |
| **E. Projection non-authoritative display (label state)** | matrix while `gba.a3.authoritative` OFF | projection `label` | display as derived view | presenting as sellable availability (P-17) | interim label visible whenever numbers are not authority-backed | never used for gating |
| **F. System/error condition** | request failed (bad context, validation, retired route) | transport/API layer | show error state | converting an error into availability 0 or hiding it | explicit error presentation | no assertion attempted |
| **G. Stale/invalid data** | **not reachable by design**: authority is `LIVE_READ` (BR-5-039); stale-by-design prohibited (TR-10.3) | — | — | representing a no-op invalidation as freshness | — | — |

### 15.2 State-transition discipline

States are derived per request/read (no stored display state). The only permitted "transitions" are changes in underlying facts: restriction edits (via §23 write path), quantity changes (assertion/room-state), resolution of previously unresolved inputs (E-1/E-3 closure changes D → A/B/C operationally). Consumers must re-read; caching a state beyond contract freshness requires TR-10.3-valid invalidation (§11.5).

## 16. Backend Consumer Contract

Legend for all rows: **Authority** = source of truth for availability decisions; **Read** = permitted reads; **Derive** = permitted derivations; **Prohibited** = forbidden calculations/behaviors; **Write** = write ownership; **Unresolved** = required behavior under `UNKNOWN`; **Migration** = Phase 5 requirement.

### 16.1 Reservations (create / confirm / guarantee / status / modify / delete / waitlist)

| Element | Contract |
|---|---|
| Authority | Assertion engine via `reservation-availability.port` inside the caller's transaction (P-6) |
| Read | `findCurrentInTransaction` only read path; snapshot for capacity consult |
| Derive | none — capacity decisions happen under lock in the engine (§5.2) |
| Prohibited | second availability leg on modify (BR-5-025 after gates); missing/random write identity (BR-5-027); treating rejection as retryable success; delete releasing availability (BR-5-042) |
| Write | Authority only (balances/movements through port); legacy `availability` leg retained **until ordered gates close** (BR-5-026/025) |
| Unresolved | `REJECTED`/`UNRESOLVED_CAPACITY` persisted; `AVAILABILITY_ASSERTION_REJECTED` (409); transaction rolls back; nothing commits (P-5) |
| Migration | create/status paths already AUTH (audit §6.1) — no change; modify dual-write removed only after §24 gates; delete gains the §26.4 rejection rule |

### 16.2 Front Office (check-in, check-out, upgrade, transfer, extend, walk-in, room options)

| Element | Contract |
|---|---|
| Authority | Room-level FO decisions (check-in options, assignment, same-type transfer) sit **outside** rate-type inventory authority (audit §13.9, preserved); capacity-affecting operations (upgrade, extend, walk-in, check-out overstay/early-departure) use the assertion port |
| Read | upgrade gate consults **authority eligibility** (BR-5-019); check-out uses `findCurrent`/`replaceInTransaction` (audit §6.2 — AUTH) |
| Prohibited | legacy `inventoryDomain.isAvailable` gating any write; its absent-row `true` default (`inventory.domain-service.ts:126`) (BR-5-019/050); direct GBA writes at check-out (zero-GBA-writes preserved, audit §13.3) |
| Write | Authority (assertion port) only; extend currently routes through `repo.update` dual-write ⇒ governed by §24 |
| Unresolved | same fail-closed as §16.1 — upgrade/extend cannot proceed on `UNKNOWN` |
| Migration | re-point upgrade gate to authority eligibility (BR-5-019) **after** BLK-P5-01 (§24 ordering) |

### 16.3 GBA (blocks, allotments, pickup, vouchers, wash/stop-sale)

| Element | Contract |
|---|---|
| Authority | GBA's own ledger (`quota − picked − released`, block counters) for entitlement; **A1 for sellable availability** (Phase 4 §3.1, preserved) |
| Read | A1 reads GBA facts as inputs; optional two-layer pickup consult (`gba.pickup.twoLayerConsult`, default OFF) reads assertion balances fail-closed |
| Derive | GBA may compute **its own** ledger remaining (contract formula) — never sellable availability |
| Prohibited | GBA publishing availability numbers (TR-1.2); pickup bypass of assertion for pickup-created reservations is a **Phase 4 carry** (A5/DEF-3) — not re-decided here (§32) |
| Write | GBA writes its ledger; Availability writes assertion state; GBA invalidation events feed A1 (TR-10.3) |
| Unresolved | two-layer consult fail-closed (balances read `UNRESOLVED` ⇒ consult refuses) |
| Migration | none re-decided; invalidation honesty governed by BR-5-039 |

### 16.4 Rates / CRS engine (quick-book, quote, book, modify, release, rates tab)

| Element | Contract |
|---|---|
| Authority | **None — legacy engine is prohibited from authority use** (§20) |
| Read | availability/restriction signals re-pointed to A1; pricing and quote-hash mechanics retained (BR-5-016) |
| Prohibited | `checkAvailability`/`assertAvailability`/`reserve`/`release` as availability truth; `quote.available` gating bookings (BR-5-050); raw `INSERT … 'CONFIRMED'` as a booking entry point (BR-5-017); unscoped reads (BR-5-051) |
| Write | booking writes only through reservation create command + assertion (BR-5-017); `crs.modifyReservation` availability leg removed after gates (BR-5-025); `availability` table writer ceases with last CRS path re-point (BR-5-018) |
| Unresolved | n/a — legacy never reports uncertainty; that is precisely why it is prohibited (§20) |
| Migration | endpoint dispositions §21.4; ordering §24; retirement schedule BLOCKED — E-5 only |

### 16.5 Operational consumers (reconciliation readers, audit-log viewers, restriction editors)

| Element | Contract |
|---|---|
| Authority | `/reconciliation` (read-only comparison) and `/availability/logs` (audit evidence) |
| Read | reconciliation may read legacy counters for comparison only; logs read audit records |
| Derive | discrepancies become **flagged facts** — never auto-repaired, never a second authority (BR-5-014, §25) |
| Write | restriction edits go through the authority-owned write path (§23); until it exists, editing affordances are visibly disabled (BR-5-041/BLK-P5-02) |
| Unresolved | a rejected write surfaces the deterministic error (§27) |
| Migration | canonical routes/payloads (§21.5) |

### 16.6 Reporting / analytics / KPI consumers (dashboards, FO KPIs, command center, reporting page)

| Element | Contract |
|---|---|
| Authority | For **availability-labelled** figures: A1 only (BR-5-028). For **occupancy**: declared reporting source (BR-5-029) |
| Read | occupancy endpoints/room-state counts as reporting inputs, hotel-scoped |
| Derive | reporting aggregation permitted server-side with declared source; **client-side occupancy formulas prohibited** (BR-5-012) |
| Prohibited | availability figures from legacy sources; mock/hardcoded values representing live truth (BR-5-029); occupancy presented as availability |
| Unresolved | display unavailable/unknown state — never fabricate |
| Migration | mislabelled computations removed or re-labelled (BR-5-028); canonical occupancy contract remains DEFERRED (P-20, §34) |

### 16.7 Channels (outbound publication)

| Element | Contract |
|---|---|
| Authority | none — push contract DEFERRED (P-19/DS-07) |
| Prohibited | publishing any availability number from a non-authority source; `channel_availability_log` caller-supplied numbers feeding UI or eligibility (BR-5-030) |
| Migration | none in Phase 5 beyond the guard (§33) |

## 17. Frontend Consumer Contract

### 17.1 Sanctioned sources

Grid/table surfaces read `GET /availability/matrix`; surfaces needing provenance/eligibility read `GET /properties/:propertyId/availability/snapshot` (§10). No other endpoint supplies availability to the frontend.

### 17.2 Shared types and hooks

One shared snapshot type + one shared projection type (BR-5-013); consumption via React Query hooks following the existing convention (`staleTime` explicit); hand-rolled payload types prohibited (REQ-10.3).

### 17.3 Eligibility surfacing (BR-5-011 — binding display duty)

For any displayed cell/figure with `bookingEligibility ∈ {ELIGIBLE, UNKNOWN, BLOCKED}` or non-empty `unresolvedSources`:

1. `UNKNOWN` and `BLOCKED` are rendered as **distinct states**.
2. When unresolved, the floor `0`/absence is shown as "unresolved/unknown" — **never** as a definite available count.
3. The interim label remains visible whenever numbers are not authority-backed (P-17).

### 17.4 Availability-labelled and occupancy figures (BR-5-028/029)

- Any header/tile/tab/total labelled available/sellable derives from A1 (through a sanctioned contract); mislabelled computations (e.g. sums of `physicalInventory` under "avail") are removed or re-labelled to what they actually compute.
- Occupancy KPIs: hotel-scoped, source-declared, never derived from a legacy availability source, never presented as availability; mock/hardcoded values may not represent live truth.

### 17.5 Affordances and canonical routes (BR-5-040/041)

- Audit-log view: `GET/POST /availability/logs` (never `/availability/matrix/logs`).
- Interval update: `POST /availability/interval-update` with backend payload `ratePlans` / `dateRange{start,end}` (never `ratePlanCodes/startDate/endDate`); paging parameters conform to the backend contract.
- Bulk update: `POST /availability/bulk-update` (already matching).
- Every availability control is either **wired to its canonical contract** or **explicitly hidden with a declared status** — silent 404 in the primary availability screen is not an acceptable steady state.
- Restriction-editing controls may only be wired to the §23 authority write path; until then they are visibly disabled (BLK-P5-02).

### 17.6 State/caching convention

Snapshot data is held in React Query cache with explicit keys; it may **not** be mirrored into a legacy Zustand store in a way that creates a second truth (§26 duty 8, audit §5.4 confirmed no snapshot store exists today — preserved). Frontend env exposes no availability flag (audit §5.6); flag state is backend/ops-owned (§28).

### 17.7 Per-surface duties

| Surface | Duty |
|---|---|
| Availability grid page | read projection; remove client occupancy/`totalAvail` formulas (F-09); surface eligibility + label |
| Quick-book rate matrix | map projection cells (replace literal `available: 0`, F-10); gate bookings on authority eligibility, never `quote.available` |
| Rates-inventory tab | stop presenting `GET /rates/availability` as "Availability Snapshot" once retired/de-availability'd (BR-5-015) |
| Dashboards / FO KPI / command center / reporting | occupancy from declared reporting source; availability-labelled chips from A1; no mocks as live truth |
| Group/allotment views | display GBA ledger values from server read models — **no client `quota − picked − released` recomputation** (BR-5-012; audit M-21) |
| Admin / mobile | greenfield: build on the canonical read contract only (never on legacy endpoints) |
| Dead artifacts | removal is Stage 4 hygiene, gated after behavior-preserving work (G-12) |

## 18. Frontend Domain Boundary

### 18.1 Owned by backend/domain

Availability computation; combination of the four fact kinds; restriction evaluation and precedence-free conflict handling; eligibility determination; quantity/capacity arithmetic; assertion enforcement; property scoping and authorization; error/unresolved semantics; freshness/invalidation; write ownership; contract shape (shared types).

### 18.2 Owned by frontend/presentation

Rendering states distinctly; grouping, formatting, ordering; display-only bed-type partition under the conservation law (§12.4); routing between the two sanctioned contracts; surfacing provenance/eligibility/labels; presenting errors and unknown states honestly.

### 18.3 Frontend MUST NOT (domain constraints — normative)

1. Independently calculate authoritative availability/sellable/occupancy-labelled figures (BR-5-012).
2. Recreate restriction logic or combine selling permission with quantity (BR-5-008).
3. Infer canonical availability from legacy values, mocks, or hardcoded defaults.
4. Silently convert `UNKNOWN`/`UNRESOLVED` into zero (or render the floor 0 as certainty) (P-3, BR-5-011).
5. Treat `UNKNOWN` as `ELIGIBLE` or gate bookings on non-authority signals (P-3, BR-5-050).
6. Create competing business rules or competing payload types (BR-5-013).
7. Bypass property isolation (BR-5-031/032/047).
8. Treat the matrix projection as a second authority (§9.3).
9. Hold snapshot data in a store that becomes a second truth (§17.6).
10. Ship a control that silently 404s (BR-5-041).

## 19. Other Consumer Migration Rules

Migration **rules** (not tasks); sequencing/implementation belongs to Stage 4 under the gates in §24.

| # | Rule | Governing input |
|---|---|---|
| M-1 | A display surface may present availability only from a sanctioned contract, or carry the interim label + eligibility surfacing while not authority-backed | BR-5-010/011, P-17 |
| M-2 | Re-pointing a **write-gating** consumer (quick-book gate, FO upgrade gate, CRS book) requires the authority to be operational first (BLK-P5-01 closed) | §24.2, BR-5-017/019 |
| M-3 | Removing a legacy **read** consumer does not wait for booking-path gates, but must respect E-5 for external consumers | §21.4, E-5 |
| M-4 | The A3 matrix migrates by value-sourcing (projection), not by endpoint replacement; its restriction reads migrate via §23 | BR-5-022/023, T-38 |
| M-5 | Occupancy surfaces migrate to declared reporting; availability-labelled surfaces migrate to A1 | BR-5-028/029 |
| M-6 | GBA/ledger views migrate to server read models (no client recomputation) | BR-5-012, audit M-21 |
| M-7 | Admin/mobile greenfield builds only on the canonical read contract | §10 |
| M-8 | Surfaces that cannot yet migrate are declared (hidden-with-status or labeled), never silently broken | BR-5-041 |
| M-9 | No migration step may invert the §24 ordering or flip a flag inside a code change | BR-5-035/037, G-5 |
| M-10 | Hygiene removals (dead components, duplicate routes) occur only **after** the behavior-preserving replacement they depend on | G-12, audit §14 B-8 |

## 20. Legacy Availability Source Disposition

| Legacy source | Classification | Domain reason | Rule |
|---|---|---|---|
| `GET /rates/availability` (ad-hoc, unscoped) | **RETIRE** (or strip all availability/occupancy fields; any retained count hotel-scoped) | not an availability computation; tenant exposure; div-by-zero prone | BR-5-015/051 |
| `GET /rates/engine/availability`, `GET /rates/engine/restrictions` | **RETIRE** as availability sources | second signals over legacy counters/raw restrictions | BR-5-016 |
| `GET /rates/engine/quote` availability signal | **RE-POINT** to authority eligibility; pricing/hash retained | signal gates bookings today (F-07) | BR-5-016/050 |
| `POST /rates/engine/book` | **RETIRE** as booking path | bypasses assertion (raw `INSERT 'CONFIRMED'`) | BR-5-017 |
| `POST /rates/engine/modify` | **GOVERNED BY §24** (its legacy leg is the dual-write leg) | drift-by-design | BR-5-025/026 |
| `POST /rates/engine/release` | **RETIRE** once no booking path writes legacy counters | a counter decrement is not an availability release | BR-5-018 |
| Legacy `availability` table + `inventory.domain-service` | **RETAIN read-only (Phase 11)**; last writer disappears with last CRS path re-point; no new readers; never authoritative | P-16 | BR-5-018 |
| FO upgrade legacy gate (`isAvailable`) | **RE-POINT** to authority eligibility; absent-row `true` default never gates a write | fail-open default | BR-5-019/050 |
| A3 raw availability computations (matrix flag-OFF path) | **PROHIBITED from authority use**; replaced by projection values when flag ON; interim labeled view allowed only while labeled | second engine; P-17 | BR-5-008/010 |
| A3 `available` overlay (`hasZeroSell ? 0 : …`) | **REMOVE** (second combination site) | INV-18 | BR-5-008/023 |
| A3 raw restriction **reads** for display | **RE-POINT** to authority projection (read side of DS-04) | filters live in one place | BR-5-022 |
| A3 raw restriction **writes** | **RE-POINT** to authority-owned write path; post-cutover raw writes prohibited | write ownership | BR-5-021 |
| `restrictions` (`rate_code='CUTOFF'`) writer | **DEFERRED — excluded from authority path** until meaning evidenced | undefined semantics, unread table | BR-5-024, E-4 |
| `room_inventory` readers (`availableRooms`) | **PROHIBITED** as availability display; table treated as legacy | no writer; unproven population | BR-5-045/046 |
| Client-side formulas (audit §5.3) | **REMOVE or RE-POINT** to server values | independent computation | BR-5-012 |
| `GET /analytics/occupancy` and FO/dashboard ad-hoc occupancy | **RETAINED as reporting only** (never availability, source-declared, hotel-scoped) | occupancy ≠ availability | BR-5-029 |
| `channel_availability_log` writes | **RETAINED, explicitly non-authoritative** (feeds nothing) | push contract deferred | BR-5-030 |
| Quick-book literal `available: 0` | **RE-POINT** to projection cells | broken display | BR-5-011/012 |
| Dead artifacts (`AvailabilitySales.tsx`, `AnimatedRoomRack.tsx`, `operasales/**`, dead DI, orphan adapters) | **REMOVE — Stage 4 hygiene, gated after behavior-preserving work** | F-21 | G-12 |

Compatibility is **never** a reason to retain a competing number (Stage 2 §5; G-4).

## 21. API Domain Contract

### 21.1 Authority surface is fixed

Exactly `GET /properties/:propertyId/availability/snapshot` and `GET /properties/:propertyId/availability/reconciliation` (P-1; plan §8 discipline `14_…:813-840`). No additional availability endpoints may be created for convenience (REQ-10.4).

### 21.2 Matrix projection

`GET /availability/matrix` — shape preserved; additive eligibility fields permitted; values from authority when flag ON; interim label while OFF (§12).

### 21.3 Restriction routes (DS-04 read side)

`GET /availability/restriction-rows` and any restriction display data must ultimately source the authority projection (BR-5-022); raw restriction reads for display are removed at the DS-04 read cutover. Restriction **writes** (`bulk-update`, `interval-update`) become callers of the §23 authority write path — their request payloads remain operational inputs, not authority bypasses.

### 21.4 Endpoint dispositions (normative — Stage 2 §10.2 carried verbatim as requirements)

| Endpoint | Domain requirement |
|---|---|
| `GET /rates/availability` | **RETIRE** (or de-availability the payload); retained counts hotel-scoped (BR-5-015) |
| `GET /rates/engine/quote` availability/blockReason signal | **RE-POINT** signal to authority eligibility; pricing retained (BR-5-016) |
| `GET /rates/engine/availability`, `GET /rates/engine/restrictions` | **RETIRE** as availability sources (BR-5-016) |
| `POST /rates/engine/book` | **RETIRE** as booking path; booking via reservation create + assertion (BR-5-017) |
| `POST /rates/engine/modify` | governed by §24 (BR-5-025) |
| `POST /rates/engine/release` | **RETIRE** once legacy-counter writers are gone (BR-5-018) |
| FO upgrade gate | **RE-POINT** to authority eligibility (BR-5-019) |
| legacy `availability` table endpoints | no new readers; retained read-only (BR-5-018) |

Retirement means the route may return 404 — compatibility aliasing is explicitly **not** required (precedent T-39 `14_…:566-568`; G-4). Retirement **timing** for external consumers waits on E-5 (§31).

### 21.5 Canonical restriction UI contract (DS-11 part 1)

| Route | Contract |
|---|---|
| `GET/POST /availability/logs` | canonical audit-log routes; frontend conforms from `/availability/matrix/logs` (BR-5-040) |
| `POST /availability/interval-update` | canonical; payload `ratePlans` + `dateRange{start,end}`; frontend conforms from `ratePlanCodes/startDate/endDate` and from the `/activities/...` prefix |
| `POST /availability/bulk-update` | canonical; already matches frontend |
| Paging/query params | reconcile to backend contract (BR-5-040) |
| Wiring condition | restriction-editing affordances wire to §23 path only; until then visibly disabled, never 404 (BLK-P5-02) |

### 21.6 Error contract and validation duty

- Deterministic machine-readable codes; consumers match **codes, never message substrings** (BR-5-020): `AVAILABILITY_ASSERTION_REJECTED` with `rejection.code ∈ {UNRESOLVED_CAPACITY, CAPACITY_BLOCKED, INSUFFICIENT_CAPACITY, …}`; plus Phase 2 outcomes (§5.2).
- **DTO duty:** no availability endpoint accepts unvalidated `body: any` for writes (Stage 2 §27.6) — availability-bearing inputs carry typed validation.
- HTTP-level realization details are Stage 4's; the **domain semantics** in §27 are fixed here.

### 21.7 Route ownership

A route is declared by exactly one controller; the duplicate `GET /tax-rates` is resolved by naming one canonical owner **after runtime verification (E-7)** and removing the other; no aliasing (BR-5-044).

### 21.8 Isolation and gateway

Every contract above obeys §22. Gateway: no availability-specific routing/versioning exists and none is invented; the catch-all stays (audit §7.5, preserved).

## 22. Property / Hotel Isolation

**REQ-22.1 (required context).** Every availability read/write requires an explicit property context: route `:propertyId` for authority/contract routes; validated hotel identity for internal operations. Absent or `hotelId === 'default'` is **rejected** at every authority entry (BR-5-032; `availability-snapshot.service.ts:18-21`).

**REQ-22.2 (scoping).** Every rule, query, projection, and write is hotel-scoped; no availability computation aggregates across hotels (BR-5-031); multi-property authorization does not exist — all Phase 5 rules are single-hotel (P-15, TR-14.3).

**REQ-22.3 (missing/invalid context behavior).** Missing context ⇒ deterministic rejection (no actor ⇒ 403 on authority routes; invalid DTO ⇒ validation error). Invalid context (non-member property, insufficient permission codes) ⇒ 403. Never: defaulting to another property, silently using header scope when route scope differs, or falling back to `'default'` (§27.1/27.2).

**REQ-22.4 (route-property authority — BR-5-047).** Where a route declares `:propertyId`, that value is authoritative for **both** authorization and data access, validated against the principal's membership; header/tenant-derived scope may not silently select a different property (closes the F-19 divergence class).

**REQ-22.5 (no guard opt-out — BR-5-048).** `@PropertyScope(false)` controllers (`availability-sales.controller.ts:30-31`, `banquet-refs.controller.ts`) may not read or write availability-bearing data unless they perform equivalent property-scoped authorization (the snapshot `authorize()` pattern: membership + permission codes + `default` rejection, `availability-snapshot.service.ts:17-33`).

**REQ-22.6 (raw SQL — BR-5-049).** Every raw statement touched by Phase 5 work includes `hotel_id` in its predicate; bare-id mutation prohibited (TR-14.1, INV-1).

**REQ-22.7 (unscoped legacy — BR-5-051).** Any legacy read found unscoped (e.g. `GET /rates/availability` F-06) is retired or scoped **regardless of consumer impact** — unscoped behavior never becomes acceptable because it currently ships (G-4).

**REQ-22.8 (leakage prevention).** Cross-property availability leakage is prevented by 22.1–22.7 combined with the projection contract (matrix/snapshot reads are property-bound); no endpoint publishes cross-hotel availability aggregates.

## 23. Restriction Write Ownership (DS-04)

**REQ-23.1 (owner).** The Availability authority **owns restriction state** as a first-class selling-permission fact. Selling permission is one of the four fact kinds combined only by A1 (P-11); its write path therefore belongs to the authority.

**REQ-23.2 (write-path requirements — BR-5-021).** Restriction writes MUST:

1. be **hotel-scoped** (predicate carries `hotel_id`; §22),
2. be **payload-validated** (typed DTO; no `body: any`; §21.6),
3. **invalidate** affected availability facts (TR-10.3 analogue for selling permission — no stale-by-design),
4. record **provenance/audit** (who/when/what changed), and
5. be **parameterized** (closes the F-26 write-side interpolation surface).

After the DS-04 cutover, raw `$executeRawUnsafe` writes to restriction tables are **prohibited**.

**REQ-23.3 (consumers may only consume).** Activities/A3 and every other UI keeps its operational screens but becomes a **caller** of the authority write path; it may no longer own restriction state. Restriction **reads for display** come from the authority projection (BR-5-022); raw display reads (`:384-405`) are removed at the DS-04 read cutover.

**REQ-23.4 (synchronization meaning).** "Synchronized" means: a successful write is reflected in subsequent snapshot/projection reads (via invalidation/recompute within the read path), with provenance attached. No asynchronous second copy of restriction state exists.

**REQ-23.5 (competing writers).**

| Phase | State | Rule |
|---|---|---|
| Interim (now) | A3 raw writers exist; authority path does not | Status quo preserved deliberately; the UI affordance that would extend raw writes is not wired (BLK-P5-02); no new raw writers |
| Cutover | Authority path becomes the only writer | All operational restriction writes route through it; A3's overlay (BR-5-023) removed in the same authority enablement window |
| Post-cutover | Single writer | Any raw restriction write is a contract violation (BR-5-021); `restrictions` (`CUTOFF`) writer is **excluded** until E-4 (BR-5-024) — it does not gain authority status by remaining |

**REQ-23.6 (canonical state cannot be established).** If the authority cannot evaluate/store selling permission for a scope (store unreadable, unproven input, conflict), the state is reported `UNRESOLVED` with provenance (§13.4) — never guessed, never silently treated as "no restriction" for unproven stores (BR-5-046).

## 24. Dual-Write / Cutover Rules (DS-05 + ordering)

### 24.1 Current transitional authority

Today the reservation modify path writes **both** the assertion engine (`replaceReservationAssertion`, `reservation.repository.ts:621`) and the legacy counters (`crs.modifyReservation`, `:625`) in one transaction, while status changes write authority-only (`:1000-1055`) — the stores **drift by design** (F-04). This dual-write state is **preserved deliberately** until the gates below close (BR-5-026): removing the writer first would strand legacy readers (§21.4 consumers still live).

### 24.2 Required ordering (verbatim — S3R-G5 / Stage 2 §22, §28)

> **DS-01 operational → DS-03 endpoint dispositions → DS-05 write-leg removal.**
> Authority enablement (B-1: DS-01 + DS-12 evidence) precedes read-consumer cutover, booking-path retirement, and write convergence (B-4, highest-risk boundary). No step may be skipped.

Derived sub-orderings (evidence-backed, Stage 2 §22):

| Step | Precondition |
|---|---|
| DS-02 execution (frontend cutover to authority-backed numbers) | DS-01 operational (E-1 + E-3 + evaluator) |
| DS-03 read retirement (BR-5-015/016) | may proceed in parallel (read side) |
| DS-03 booking retirement (BR-5-017/019) | operational authority |
| DS-05 leg removal (BR-5-025) | DS-01 **and** DS-03 dispositions landed |
| `gba.a3.authoritative` ON (BR-5-037) | evaluator + E-1 + E-3 + parity soak |
| DS-11 part 1 wiring (BR-5-040/041) | DS-04 authority write path exists (BLK-P5-02) |

### 24.3 No premature cuts (BR-5-026)

While the authority is not operational: **no write-path reduction of any kind** — no partial cuts, no "temporary" removal of a legacy leg, no flag-ON experiments with the stub bound (§13.7).

### 24.4 Cutover conditions and writer behavior

| Actor | Before cutover | After its gate closes |
|---|---|---|
| Legacy writer (`crs.modifyReservation` availability leg; `inventory.domain-service` reserve/release) | runs as today (dual-write) | removed: modify writes availability **only** through the assertion port (BR-5-025); legacy counter writer ceases when the last CRS booking/modify path is re-pointed (BR-5-018) |
| Canonical writer (assertion engine) | already authoritative for create/status/cancel paths | sole write truth per fact (single-writer principle, plan §11 "no dual-write") |
| Legacy readers (rates tab, upgrade gate, quick-book) | still read legacy | re-pointed/retired per §21.4 **before** the writers are removed |
| Legacy `availability` table | read-write (single writer file) | read-only, retained to Phase 11 (P-16) |

### 24.5 Write identity (BR-5-027 — preserved)

Every availability-bearing write carries deterministic operation identity: explicit `idempotencyKey` where the caller has one (HTTP `Idempotency-Key` / channel upstream identity / scheduled durable per-item identity — D-15); **never** `hotelId || 'default'` (authority rejects `'default'`); never an absent identity that silently mints `randomUUID()` operation keys; `actorId` never silently `'system'` where a real actor exists (disposes F-16).

### 24.6 Verification requirements

- Drift between legacy counters and authority is observed through `/reconciliation` (§25), read-only, reported as flagged facts.
- Each cutover step is verified by executed evidence in its stage (BR-5-033) and by the deterministic absence/parity expectations stated in §30 (AC-23…AC-25, AC-32).
- Flag state changes carry recorded gate evidence (§28.2).

### 24.7 Rollback / safety boundary (already decided)

- Every switch is a flag or a redeploy: `gba.a3.authoritative` OFF restores labeled matrix; the unresolved stub binding remains available as rollback artifact (§13.7); flags default OFF (P-21).
- Legacy table deletion is **out of scope** (Phase 11) — rollback never requires schema change (G-1).

### 24.8 MUST NOT happen before prerequisites

1. No consumer re-pointed onto the authority for **write gating** before BLK-P5-01 closes (§13.7).
2. No `crs.modifyReservation` leg removal before DS-01 + DS-03 (§24.2).
3. No restriction-editing affordance wired to raw writes (BLK-P5-02).
4. No flag flips inside code changes (G-5).
5. No `gba.a3.authoritative` ON while the stub is bound (BR-5-009/037).

## 25. Reconciliation Contract

**REQ-25.1 (sanctioned comparison).** `GET /properties/:propertyId/availability/reconciliation?roomType&arrivalDate&departureDate` is the **only** sanctioned legacy-vs-authority comparison (BR-5-014); read-only.

**REQ-25.2 (duties).**

1. Discrepancies are **flagged facts**, never auto-repaired (TR-10.4, `13_…:745-751`).
2. **Legacy never wins**: comparison exists for migration readiness and drift visibility; when authority and legacy differ, the authority value is the domain truth and the legacy value is the non-authoritative projection (TR-15.5).
3. Dual-write drift (F-04) is monitored here until §24.4 removes the divergence.
4. Assertion integrity: balances are the sum of movements (append-only trigger §5.2); any attempt to "fix" a balance directly violates the schema contract — reconciliation reports, never repairs.

**REQ-25.3 (boundary).** Reconciliation is **evidence**, not authority: it may not feed availability decisions, eligibility, or publication; it may not write any domain state.

## 26. Canonical Logs / Audit Contract (DS-11)

### 26.1 Canonical log routes

`GET/POST /availability/logs` are the canonical audit-log routes (BR-5-040); `/availability/matrix/logs` is not a contract. Log creation/reading is scoped like any availability data (§22).

### 26.2 Four distinct layers (never conflated)

| Layer | What it is | Examples | Authority? |
|---|---|---|---|
| **Domain state** | the current business truth | assertion balances/movements, `reservation_availability_state`, restriction state (post-§23), snapshot values | **yes** (only layer that is) |
| **Audit evidence** | append-only record of operations/decisions | `availability_assertion_movements` (trigger-enforced append-only), `reservation_availability_operations` journal, rows served by `/availability/logs` | no — evidences domain state |
| **Reconciliation evidence** | comparison output | `/reconciliation` deltas, GBA detectors (`gba.reconciliation.enabled`, `automaticRepair:false`) | no |
| **Operational logs** | runtime traces | application logs, consumer retry/error logs, monitoring gates | no |

**A log may never become a second business authority**: no availability figure may be sourced from logs, and no log write may alter domain state (G-8 family).

### 26.3 Availability-state ownership

Availability state rows (`reservation_availability_state`, `reservation_availability_operations`, `availability_assertions`, `availability_assertion_balances`, movements) are **owned exclusively by the Availability/assertion engine**. No reservations-side or Activities-side path may create, update, or delete them outside the port contract (Phase 4 §3.1 fact ownership; audit §13.14 verified compliant — preserved).

### 26.4 Delete behavior (DS-11 part 2 — decided)

1. Hard-delete remains **terminal-only** (Decision 12, preserved) and performs **no availability operation**: no release, no balance change, capacity attributable to a completed stay is retained (BR-5-042; P-8/P-9).
2. If `reservation_availability_state` / `reservation_availability_operations` rows exist for the reservation, delete is **rejected deterministically with a typed business error before any mutation** — no partial mutation, no raw FK error surfacing as a 500 (BR-5-043; schema `onDelete: Restrict` intent `schema.prisma:17399,:17426`).
3. Assertion movements/audit rows are **never cascade-deleted** from a reservations path (G-10); balances (`(hotel_id, room_type, stay_date)`, no reservation FK) and free-text movement `reference_id` survive independently.
4. Operator guidance: terminal reservations with availability history are retained.

### 26.5 Auditability

Every availability-bearing mutation carries deterministic identity (§24.5), actor provenance, hotel scope, and affected dates — sufficient to reconstruct who changed what from the audit layer alone.

## 27. Error / Failure Semantics (domain level)

| Condition | Domain behavior | Contract behavior (required shape) |
|---|---|---|
| Missing property context / no actor | request rejected; no default | deterministic 403-class rejection on authority routes; never `hotelId='default'` fallback |
| Invalid property context (non-member, insufficient permissions, route≠authorized property) | request rejected | 403-class rejection; route-property authority applies (§22.4) |
| Unresolved capacity at assert | nothing commits; `REJECTED` persisted with `UNRESOLVED_CAPACITY` | `AVAILABILITY_ASSERTION_REJECTED` (409) with `rejection.code=UNRESOLVED_CAPACITY`; consumers match codes |
| Blocked by restriction at assert | rejected with blocked semantics | `rejection.code=CAPACITY_BLOCKED` family |
| Insufficient capacity (actual zero / under floor) | rejected; capacity unchanged | `rejection.code=INSUFFICIENT_CAPACITY` |
| Restriction inputs conflicting | dimension `UNRESOLVED` + `sourceConflicts` ⇒ outcome `UNRESOLVED` ⇒ fail-closed | snapshot surfaces `UNKNOWN`; no special "conflict" success path |
| Restriction store unreadable | dimension `UNRESOLVED` with reason | as above (BR-5-001 error behavior) |
| Stale data | **not applicable by design** (`LIVE_READ`); future cache must implement TR-10.3 invalidation | no stale-success contract exists |
| Invalid request (bad dates, malformed payload, unvalidated write body) | validation failure before any domain effect | deterministic validation error; DTO duty (§21.6) |
| Retired endpoint (§21.4) | route absent | **404 acceptable** — retirement, not compat alias (T-39 precedent) |
| Forbidden authority path (write through read contract, raw write post-cutover, second combination) | contract violation | rejected at the boundary; surfaced as deterministic error |
| Idempotency | same key + same hash ⇒ replay of recorded result; different hash ⇒ conflict | `IDEMPOTENCY_CONFLICT` (409) |
| Concurrent availability mutation | one wins; loser fails with conflict or retries; no merge | `CONFLICT` / `OPERATION_IN_PROGRESS` (409); no silent last-write-wins |
| Consuming state without assertion | refusal | `CURRENT_ASSERTION_MISSING` (409) |
| Release/replace against wrong assertion | refusal | `ASSERTION_LINK_MISMATCH` |
| Delete of terminal reservation with availability state | rejected before mutation (§26.4) | deterministic typed business error; no raw FK error |
| Duplicate route declaration | single owner after E-7; other declaration removed | no aliasing (BR-5-044) |
| Consumer detects failure by message substring | prohibited | must match `AVAILABILITY_ASSERTION_REJECTED` / `rejection.code` (BR-5-020) |

Fail-closed is the **default posture**, not an error path (P-5): when capacity cannot be safely resolved, nothing commits.

## 28. Environment / Feature Flag Contract (DS-10)

### 28.1 Flag inventory (env `FEATURE_*`, default OFF, read via `ConfigService.getFeatureFlag`)

| Flag | What it controls | Phase 5 relevance |
|---|---|---|
| `gba.a3.authoritative` | matrix numbers: authority-backed vs raw/legacy computation | **the Phase 5 authority-display gate** — ON only per §28.3 |
| `gba.consumers.cascade` | reservation→pickup cascade + GBA invalidation | **MUST be ON at deploy** (P-21, `17_…:80`) — preserved |
| `gba.wash.schedulerEnabled` | hourly wash + `washMetadataAuthoritative` | BLOCKED on Deviations A+B (P-21) — untouched |
| `gba.reconciliation.enabled` | hourly detectors | soak tooling; read-only evidence (§25) |
| `gba.pickup.twoLayerConsult` | pickup read-time consult vs quota-only guard | default OFF; GBA-side, not re-decided |
| `gba.pickup.canonicalRead` | pickup read source | rollout sequence preserved: enable first |
| `gba.pickup.canonicalWrite` | legacy ledger freeze | enable second; sequence preserved |

### 28.2 Single mechanism and documentation duties

- The seven env `FEATURE_*` flags are the **only** mechanism for availability rollout; **no availability flag may exist on the platform DB flag system** (`platform/configuration/feature-flag.service.ts`), and none does today (BR-5-035).
- Flag state changes are **operational actions with recorded gate evidence — never a side effect of a code change** (BR-5-035, G-5).
- All seven flags (including `gba.consumers.cascade`) must be declared in repository config documentation (`compose.yaml` / `.env.example`) — a **Stage 4 configuration duty** decided now (BR-5-036).

### 28.3 What `gba.a3.authoritative` gates, and its ON gate

**Controls:** whether `GET /availability/matrix` serves authority-sourced values (§12.2) or the labeled legacy view. **Affects:** A3 matrix consumers (availability page, any matrix reader).

**ON gate (BR-5-037 — operational closure of BLK-P5-01), all required:**

1. restriction evaluator implemented (DS-01.2) over the evidenced store set (§13.2),
2. input population evidence received (**E-1**),
3. runtime confirmation of the production chain (**E-3**),
4. parity soak green.

Until all four hold, the flag **must remain OFF** (BR-5-009): ON with the unresolved stub would zero every matrix cell (`:540`).

### 28.4 What flags MUST NOT control

- Business rules or eligibility semantics (no rule changes state based on a flag; flags select **sources/paths**, never truth).
- Freshness claims: today's `cache.delPattern` no-op is not a freshness mechanism; any future cache must implement TR-10.3 for real (BR-5-039, F-14).
- Test evidence: flags do not substitute for executed runs (BR-5-033).

### 28.5 Preserved rollout invariants (P-21)

`gba.consumers.cascade` ON at deploy; wash blocked by Deviations A+B; `canonicalRead` → soak → `canonicalWrite` untouched; all other flags default OFF; Phase 5 adds **no** flag beyond documenting the existing seven (no new availability flags introduced by this specification).

## 29. Domain Invariants

Specification-level invariants (binding; each testable via §30). `P-n`/`BR-5-nnn`/`INV-n`/`TR-*` cite authority.

| # | Invariant | Source |
|---|---|---|
| INV-P5-01 | **Single availability authority:** exactly one producer of sellable availability (A1); no second sellable number exists anywhere, including rebuilt A3 | P-12, INV-18, BR-5-008 |
| INV-P5-02 | **Single combination:** only A1 combines the four fact kinds; no component outside A1 combines selling permission with quantity | P-11, BR-5-008 |
| INV-P5-03 | **Snapshot is read-only;** all availability mutations occur through the Phase 2 assertion port inside the caller's transaction (synchronous, fail-closed) | P-1, P-6, BR-5-010 |
| INV-P5-04 | **Fail-closed:** any unresolved source/dimension ⇒ `physicalAvailable=0`, `sellableAvailable=0` (floor), `bookingEligibility='UNKNOWN'`, consumption figures `null`, `unresolvedSources` surfaced | P-3, BR-5-005 |
| INV-P5-05 | **`UNKNOWN` is never `ELIGIBLE`** — no consumer (backend or frontend) may treat or display it as permission to sell | P-3, BR-5-011 |
| INV-P5-06 | **Nothing commits when capacity cannot be resolved:** assertion persists `REJECTED`/`UNRESOLVED_CAPACITY` and the transaction rolls back; fail-closed is the default, never an error path | P-5, P-6, P-7 |
| INV-P5-07 | **Arithmetic floors:** every availability quantity is `max(0, …)`; `sellLimit` can only tighten capacity; overage is measured, never silently clamped | P-4, BR-5-006 |
| INV-P5-08 | **Stop sale never changes quantity or releases rooms;** evaluation order: stop sale before remaining; lift restores display only | P-14, INV-17, BR-5-007 |
| INV-P5-09 | **Outcome status integrity:** `RestrictionOutcome.status='RESOLVED'` iff every applicable dimension is `RESOLVED`/`NOT_APPLICABLE`; any `unresolvedSources` entry forbids `RESOLVED` | BR-5-004 |
| INV-P5-10 | **Conflict ⇒ `UNRESOLVED`:** disagreeing stores yield `UNRESOLVED` + `sourceConflicts`; no precedence ranking may be invented or silently applied | BR-5-003, E-2, G-3 |
| INV-P5-11 | **Absence and provenance:** a store without a row imposes nothing (recorded); a store without recorded writer/population is flagged, never silently trusted | BR-5-002, BR-5-046 |
| INV-P5-12 | **Single restriction writer (post-cutover):** restriction state is written only through the authority path (scoped, validated, invalidating, audited); raw restriction writes prohibited after cutover | BR-5-021 |
| INV-P5-13 | **Hotel isolation:** every read/write — including raw SQL — carries `hotel_id`; bare-id mutation prohibited; no cross-hotel availability aggregation | P-15, INV-1, BR-5-031/049 |
| INV-P5-14 | **Context integrity:** missing context or `hotelId==='default'` rejected at every authority entry; route `:propertyId` is authoritative for authorization **and** data access | BR-5-032/047 |
| INV-P5-15 | **Two sanctioned contracts:** exactly `/snapshot` (canonical) and `/matrix` (projection), plus `/reconciliation` (comparison); no third availability contract | BR-5-010, P-1 |
| INV-P5-16 | **Projection fidelity:** matrix shape preserved; values authority-sourced when flag ON; interim label present while OFF; additive eligibility fields permitted only | P-17, P-18, BR-5-010/011 |
| INV-P5-17 | **Bed-type conservation:** Σ bed-type cells = authority room-type figure per date; residue deterministic and disclosed | BR-5-012 |
| INV-P5-18 | **No client-side availability computation;** occupancy/availability-labelled figures derive from server authority/reporting with declared source | BR-5-012/028/029 |
| INV-P5-19 | **Eligibility must be surfaced:** `ELIGIBLE`/`UNKNOWN`/`BLOCKED` rendered distinctly; unresolved floor never shown as certainty; label visible while non-authority-backed | BR-5-011 |
| INV-P5-20 | **One shared contract type** for snapshot and one for projection; hand-rolled duplicates prohibited | BR-5-013 |
| INV-P5-21 | **No legacy signal gates or publishes availability** (rates engine, `quote.available`, FO `isAvailable`, quick-book gate) | BR-5-050, BR-5-019/017 |
| INV-P5-22 | **Delete never releases:** terminal-only hard-delete performs no availability operation; delete with availability state is rejected deterministically; audit/balance rows never cascade-deleted | P-8, P-9, BR-5-042/043, G-10 |
| INV-P5-23 | **Movements are append-only;** balances equal the sum of movements; no path sets a balance directly | Phase 2 §5.1/§5.2 (extract `:130-155`) |
| INV-P5-24 | **At most one live assertion per reservation per property** (`ACTIVE`/`PARTIALLY_RELEASED`); release/replace target exact assertion ids | Phase 2/3 (`extract :149`, `:201`) |
| INV-P5-25 | **Deterministic write identity:** explicit idempotency keys; never `'default'` hotel; no silently minted random operation keys; replay returns recorded result | BR-5-027, Phase 2 §5.3/§5.4, D-15 |
| INV-P5-26 | **Ordering gates:** DS-01 operational → DS-03 dispositions → DS-05 leg removal; no write-path reduction before those gates; no flag-ON with the stub bound | BR-5-025/026/009 |
| INV-P5-27 | **Flag integrity:** single env `FEATURE_*` mechanism; defaults OFF; ON only with recorded gate evidence; no code-level flips; `gba.consumers.cascade` ON at deploy | BR-5-035/037/038, P-21, G-5 |
| INV-P5-28 | **Reconciliation is read-only evidence:** discrepancies flagged, never auto-repaired; legacy never wins | BR-5-014, TR-15.5, TR-10.4 |
| INV-P5-29 | **Logs are never authority:** audit/reconciliation/operational layers cannot feed availability decisions or alter domain state | §26.2, G-8 family |
| INV-P5-30 | **No outbound availability publication from non-authority sources;** existing channel log stays explicitly non-authoritative | BR-5-030 |
| INV-P5-31 | **`room_inventory` is not an availability source;** no figure sourced solely from it may be displayed or returned as availability | BR-5-045 |
| INV-P5-32 | **Deterministic error contract:** consumers detect assertion/overbooking failures by code (`AVAILABILITY_ASSERTION_REJECTED`, `rejection.code`), never message substrings; one route has exactly one owner | BR-5-020/044 |
| INV-P5-33 | **Freshness honesty:** authority is `LIVE_READ`; no cache exists today; stale-by-design prohibited; future caches implement TR-10.3 invalidation for real | BR-5-039, TR-10.3 |
| INV-P5-34 | **State distinctions preserved:** actual zero, blocked, unknown/unresolved, projection label, and error are distinct states that may not be collapsed into a single number | §15, P-3 |

## 30. Acceptance Conditions

Specification-level acceptance conditions (Given → When → Then). These state observable domain outcomes, **not** test implementations.

### 30.1 Authority selection and quantity semantics

- **AC-01 (single truth).** *Given* two consumer surfaces show availability for the same `(hotel, roomType, date)`, *when* both are read at the same freshness, *then* their values are identical modulo the sanctioned bed-type partition, and every value traces to the A1 snapshot for identical inputs.
- **AC-02 (no second combination).** *Given* the A3 matrix with `gba.a3.authoritative` ON, *when* a cell is produced, *then* it equals the authority fact with **no** local restriction overlay (`hasZeroSell ? 0 : …` removed); *when* OFF, the interim label is present (AC-15).
- **AC-03 (arithmetic).** *Given* any snapshot computation, *then* no quantity is negative; `sellLimit` only lowers `sellableCapacity`; `overbookingUsed = max(0, consumption − physicalCapacity)` is reported, not suppressed.
- **AC-04 (unresolved floors).** *Given* any source unresolved, *when* the snapshot is produced, *then* `physicalAvailable=0`, `sellableAvailable=0`, `bookingEligibility='UNKNOWN'`, consumption fields `null`, `unresolvedSources` non-empty.
- **AC-05 (blocked vs unknown).** *Given* a resolved blocking restriction, *then* `bookingEligibility='BLOCKED'`, `sellableAvailable=0`; *given* an unresolved input, *then* `'UNKNOWN'` — the two states are never interchangeable.
- **AC-06 (actual zero).** *Given* all inputs resolved, no blocking restriction, and computed `sellableAvailable=0`, *then* eligibility is `ELIGIBLE` with a certain zero; an assertion attempt fails with `INSUFFICIENT_CAPACITY`.

### 30.2 Restriction resolution and unresolved capacity

- **AC-07 (read failure).** *Given* an in-scope restriction store cannot be read, *then* its dimension is `UNRESOLVED` with a reason; the outcome may not be `RESOLVED`.
- **AC-08 (conflict).** *Given* two applicable stores disagree on a dimension, *then* that dimension is `UNRESOLVED`, `sourceConflicts[]` records store+value pairs, and **no** store is silently preferred.
- **AC-09 (absence).** *Given* an in-scope store with an identified writer/population has no row for the scope, *then* the dimension is `RESOLVED` with value "no restriction" and the absence check appears in provenance; *given* the store is unproven (E-1), *then* its contribution is flagged or excluded — absence is not read as permission.
- **AC-10 (outcome integrity).** *Given* any applicable dimension `UNRESOLVED`, *then* `RestrictionOutcome.status='UNRESOLVED'`.
- **AC-11 (stop sale).** *Given* allotment stop sale applied for `(hotel,date,category)`, *then* selling display shows remaining 0, quota counters are unchanged, pickup guards still refuse intake, and lifting restores display only.
- **AC-12 (assertion rejection).** *Given* `bookingEligibility='UNKNOWN'`, *when* a consuming reservation write asserts, *then* the assertion persists `REJECTED`/`UNRESOLVED_CAPACITY`, the caller receives `AVAILABILITY_ASSERTION_REJECTED` (409) with `rejection.code=UNRESOLVED_CAPACITY`, and no partial state commits.

### 30.3 Property isolation

- **AC-13 (context).** *Given* a request without valid property context or with `hotelId='default'`, *then* it is rejected; *given* route `:propertyId` differs from header/tenant scope, *then* the route value governs both authorization and data access.
- **AC-14 (scoping).** *Given* any availability read/write, *then* its predicate includes `hotel_id`; no query aggregates across hotels; a `@PropertyScope(false)` controller touching availability data performs equivalent scoped authorization.
- **AC-15 (unscoped legacy).** *Given* a legacy availability read lacking `hotel_id` (F-06 class), *then* it is retired or scoped — consumer impact does not exempt it.

### 30.4 Frontend authority and contracts

- **AC-16 (contract set).** *Given* any new frontend availability data need, *then* it is served by `/snapshot` or `/matrix` only; no third contract exists.
- **AC-17 (projection shape/label).** *Given* the matrix with flag OFF, *then* response shape is unchanged and `label: 'derived view, not sellable availability'` is present and rendered; *with* flag ON (post-gate), values equal authority facts.
- **AC-18 (conservation).** *Given* a room type with multiple bed types and an authority value V for a date, *then* the displayed bed-type cells sum exactly to V, with deterministic, disclosed residue handling.
- **AC-19 (eligibility surfacing).** *Given* a cell with `UNKNOWN` or `BLOCKED` (or non-empty `unresolvedSources`), *then* it renders as a distinct unknown/blocked state — never a bare number — and never as permission to book.
- **AC-20 (no client math).** *Given* any frontend computation producing available/sellable/occupancy-labelled output (audit §5.3 list), *then* it is absent after migration; group/allotment views display server-provided ledger values.
- **AC-21 (shared types).** *Given* snapshot or projection data crossing the API boundary, *then* both sides reference the single shared type; no duplicate payload declaration exists.
- **AC-22 (quick-book honesty).** *Given* the quick-book matrix, *then* cells map projection values (no literal `available: 0`) and carry eligibility surfacing.

### 30.5 Legacy authority prohibition and API contract

- **AC-23 (retired/ scoped reads).** *Given* `GET /rates/availability` after disposition, *then* it is absent or returns no availability/occupancy figure; any retained count is hotel-scoped.
- **AC-24 (no legacy gate).** *Given* a booking attempt (quick-book, FO upgrade) after its gate re-point, *then* the gate consults authority eligibility; the legacy absent-row `true` default never permits a write; `quote.available` is not the gate.
- **AC-25 (booking path).** *Given* a booking after retirement, *then* it executes through the reservation create command with assertion; `/rates/engine/book` no longer inserts `CONFIRMED` rows.
- **AC-26 (canonical restriction routes).** *Given* the audit-log or interval-update UI, *then* it calls `/availability/logs` and `/availability/interval-update` with `ratePlans`/`dateRange{start,end}`; no call to `/availability/matrix/logs` or `/activities/...` remains.
- **AC-27 (error contract).** *Given* an overbooking/assertion failure, *then* consumers detect it via `AVAILABILITY_ASSERTION_REJECTED`/`rejection.code`; no `err.message.includes('inventory')`-style detection exists.
- **AC-28 (validation).** *Given* a write to any availability endpoint with an unvalidated payload, *then* it is rejected by typed validation before any domain effect.
- **AC-29 (route ownership).** *Given* a duplicate route declaration, *then* exactly one controller declares it after E-7 verification; the other declaration is removed; no alias remains.

### 30.6 Write ownership, cutover, reconciliation, logging

- **AC-30 (restriction write path).** *Given* a restriction edit after DS-04 cutover, *then* it is hotel-scoped, validated, invalidates affected availability, records provenance/audit, and no raw `$executeRawUnsafe` restriction write executes.
- **AC-31 (unproven store).** *Given* `restrictions` (`rate_code='CUTOFF'`) without evidenced meaning (E-4), *then* it is not migrated into the authority write/read path and gains no authority status.
- **AC-32 (ordering).** *Given* the authority is not operational (E-1/E-3 open), *then* no legacy write leg is removed, no write-gating consumer is re-pointed, and no flag flips; *after* DS-01+DS-03, modify writes availability only through the assertion port.
- **AC-33 (write identity).** *Given* an availability-bearing write, *then* it carries a deterministic operation identity (explicit key or D-15 derivation), never `'default'`, never a silently minted random key; replay returns the recorded result.
- **AC-34 (delete).** *Given* a terminal reservation with availability state rows, *when* delete is attempted, *then* it is rejected with a deterministic typed business error before any mutation; balances and audit rows are untouched; *given* a terminal reservation without such rows, *then* delete performs no availability operation.
- **AC-35 (reconciliation).** *Given* legacy and authority values differ, *when* `/reconciliation` is read, *then* the delta is reported as a flagged fact; nothing is rewritten; the authority value governs all domain decisions.
- **AC-36 (audit integrity).** *Given* any availability mutation, *then* an audit record with identity/actor/hotel/date exists; movements remain append-only; no reservations path deletes audit or balance rows; logs never feed availability decisions.

### 30.7 Governance (flags, evidence, scope)

- **AC-37 (flag mechanism).** *Given* an availability rollout flag, *then* it is an env `FEATURE_*` read via `ConfigService.getFeatureFlag`, default OFF, declared in config documentation; no platform-DB availability flag exists; no code change flips a flag.
- **AC-38 (authoritative gate).** *Given* `gba.a3.authoritative` ON, *then* all four BR-5-037 conditions are evidenced (evaluator, E-1, E-3, parity soak); otherwise the flag is OFF.
- **AC-39 (deploy invariant).** *Given* a deployment, *then* `gba.consumers.cascade` is ON and wash remains blocked by Deviations A+B; canonicalRead→soak→canonicalWrite sequence is untouched.
- **AC-40 (input validity).** *Given* an authority input store with no recorded writer/population, *then* its contribution carries provenance-flagged uncertainty or is excluded — never presented as confirmed fact.
- **AC-41 (no phantom publication).** *Given* the deferred push contract, *then* no endpoint publishes an availability number from a non-authority source and `channel_availability_log` feeds no UI or eligibility decision.
- **AC-42 (KPI honesty).** *Given* a dashboard/KPI figure, *then* availability-labelled values derive from A1 and occupancy is source-declared, hotel-scoped, not availability; no mock/hardcoded value is presented as live truth.
- **AC-43 (stage evidence).** *Given* a code-changing stage exit, *then* its artifact records API suite results with `AVAILABILITY_TEST_DATABASE_URL` set (0 harness-skips for that reason), web typecheck/lint/test results, and baseline reconciliation (E-8).
- **AC-44 (scope integrity).** *Given* Phase 5 work, *then* no schema/migration/test/config change occurs outside decided Stage 4 duties, no deferred item (DS-07/DS-08/occupancy contract) is partially implemented, and hygiene work does not precede its behavior-preserving replacement.
- **AC-45 (affordance honesty).** *Given* an availability-page restriction-editing affordance (audit-log view, interval update) before the DS-04 authority write path exists, *when* it is used or inspected, *then* it is visibly disabled/inert — never a silent 404 and never wired to a raw restriction write; *after* DS-04, it calls the canonical authority routes with validated payloads (BLK-P5-02).
- **AC-46 (decision integrity).** *Given* any Phase 5 change proposal, *then* no ratified Phase 1–4 decision (§4/§5 register) is re-opened without the §3 amendment protocol and new evidence; conflicts between Stage 2 decisions and earlier authority are documented in §3, not silently resolved.
- **AC-47 (interim gate).** *Given* the unresolved stub is bound (evaluator not shipped), *then* `gba.a3.authoritative` is OFF, no consumer is cut over for write gating, and the stub remains available as the rollback artifact (§13.7).

## 31. Deferred / Blocked Domain Items

**Evidence blockers are not implementation tasks** (brief §23): each row states knowledge state, affected specification, and why the affected portion cannot be finalized here.

| ID | Known | Not known | Affected section | Why it cannot finalize here | Evidence required | Blocks scope |
|---|---|---|---|---|---|---|
| **E-1** (U-3) | Input store set is defined by writers/reads found in-repo; absence semantics decided (BR-5-002) | Which stores actually carry data per hotel, recency, and who writes `room_inventory`, `out_of_order`, `out_of_service`, `rate_restrictions`, six A3 tables | §13.2/§13.3 (store list), §29 INV-11 | population facts require data inspection, not decision-making | per-hotel row counts + recency + writer identification | **BLK-P5-01 (hard)** — blocks evaluator operationalization, flag-ON, DS-02 execution, DS-03 booking retirement, DS-05 |
| **E-2** | Default handling decided: conflict ⇒ `UNRESOLVED` (BR-5-003); authority precedent exists | Whether product/operational intent defines an explicit precedence ranking (only needed if real conflicts exist) | §13.4 | inventing precedence is prohibited (G-3) | product/operational documentation of ranking intent | nothing (fail-closed default stands) |
| **E-3** (U-1) | Static chain verified: DI binds unresolved stub ⇒ snapshot `UNKNOWN` ⇒ assertion `REJECTED` ⇒ 409 (Stage 2 §6.2) | Runtime confirmation that production behaves exactly so | §13.6, §28.3 | executed observation not performed in Stages 1–3 | end-to-end create attempt observed | **BLK-P5-01** — gates operational enablement and BR-5-037 |
| **E-4** | Store written by A3 (`:878-885`); read by nothing; `zero_sell_value` domain undefined | Business meaning of `restrictions` (`rate_code='CUTOFF'`) and `zero_sell_value` | §13.2, §23.5 (exclusion), BR-5-024 | semantics undefined anywhere in corpus | product/domain definition | evaluator scope for that store only |
| **E-5** (U-5) | Dispositions fixed (§21.4); retirement is decided | Whether out-of-repo consumers of `/rates/engine/*` exist and need deprecation windows | §21.4 (schedule column) | external inventory not inspectable in-repo | external-consumer inventory | retirement **timing only** — disposition unchanged |
| **E-6** (U-6) | Rule issued: error detection by code (BR-5-020/AC-27) | Live behavior of the OTA overbooking branch (`webhook.service.ts:104`) | §27 | runtime observation not performed | live branch check | nothing (rule already issued) |
| **E-7** (U-2) | Both declarations known (`banquet-refs.controller.ts:45`, `availability-sales.controller.ts:131`); registration order known | Which controller actually serves `GET /tax-rates` at runtime | §21.7 | runtime winner unverified | runtime verification | BR-5-044 execution |
| **E-8** (U-4) | Documented baselines (API 2 suites/6 tests; web 1 suite/10 tests); static analysis (F-22) | Reconciled executed baselines | §36 (stage exits), §30 AC-43 | Stage 3 executes nothing (documented) | executed runs with DB env | first code-changing stage exit |

**Blocker classes carried forward (Stage 2 §21):** BLK-P5-01 `HARD` (E-1+E-3+evaluator) · BLK-P5-02 `CONDITIONAL` (DS-04) · BLK-P5-03 `CONDITIONAL` (ordering) · BLK-P5-04/05 `NON-BLOCKING` (evidence/flag governance) · BLK-P5-06 `DEFERRED` (scope).

**Operational prohibition while BLK-P5-01 is open:** flag OFF; no write-gating cutover; no consumer presented with authority numbers as final; stub retained as rollback (§13.7).

## 32. Phase 4 Carry-over Dependencies

Format: `Phase 4 Carry-over → Phase 5 Dependency → Affected Requirement` (no Phase 4 decision is changed).

| Carry-over (state preserved) | Phase 5 dependency | Affected requirement here |
|---|---|---|
| Deviation A (BLK-1: `GUARANTEED_BLOCK` wash exclusion not shipped; t45 skipped) — hard precondition before wash activation | none for Phase 5 decisions | §28.5 wash stays blocked (BR-5-038) |
| Deviation B (BLK-2: durable wash/release/attrition store undecided) | none | same |
| Deviation C (no schema changes; Phase 2 DB foundation skipped) | reinforces G-1 | §2.2, §33 G-1 |
| Deviation D (NB-1…NB-6 doc chores) | absorbed as Stage 4 doc duties | §36 (F-23) |
| DEF-1…DEF-10 (mechanisms, never business rules) | not re-opened | §3.2 |
| Flags & rollout sequence (P-21) | Phase 5 must not alter sequence | §28.2/28.5 |
| Test baselines (P-22) | reconciliation duty | §36 (BR-5-033, E-8) |
| A3 interim label + matrix shape preservation (P-17/P-18) | projection contract must retain both | §12.1/12.2 |
| `CHECKED_OUT` non-release; terminal-only delete; consumption adapter D-0/D-1 semantics | delete rules must clarify, not change | §26.4, §5.2 |
| Pickup ↔ assertion port shape (DEF-3/T-41; R-13 pickup-created reservations bypass assertion) | **inherited dependency of BR-5-005 enforcement** — not re-decided | §16.3, §34 (recorded, Phase 4-owned) |
| Phase 4 open actions (commit delta, canonical cutover, cascade flag config, BLK-1/2 convening — `17_…:92-99`) | Phase 5 may not assume they are done (F-24) | §34, §36 |

## 33. Explicitly Prohibited Behaviors

### 33.1 Status

These prohibitions are binding on every subsequent stage and on every consumer of this specification. They are stated as hard "must not" behavior — convenience, current-consumer pressure, and partial-evidence arguments do not lift them; only the §3 amendment protocol (new evidence, recorded) can change any of them.

### 33.2 Stage 3 guardrails (G-1…G-12, reproduced from Stage 2 §30.2)

- **G-1** No schema, migration, index, constraint, or seed changes (user constraint; Phase 4 deviation C).
- **G-2** No fail-open: never suppress `unresolvedSources`, never treat `UNKNOWN` as `ELIGIBLE`, never bypass `unassertableReason` (P-3/P-5/P-6/P-7).
- **G-3** No invented restriction policy: no precedence ranking, no invented closure/LOS/sell-limit semantics (BR-5-003; §31 E-2 is the only path to a ranking).
- **G-4** No legacy-comes-first compatibility: retiring or scoping a legacy surface may not be vetoed by its current consumers (BR-5-051).
- **G-5** No flag flips inside code changes; no new availability flags on the platform DB flag system (BR-5-035).
- **G-6** No test-execution claims without recorded runs (BR-5-033); no reliance on harness-skipped suites as evidence.
- **G-7** No cross-hotel aggregation, no `hotelId==='default'` fallback, no unscoped raw SQL (BR-5-031/032/049).
- **G-8** No third availability contract, no client-side availability math, no second combination site (BR-5-008/010/012).
- **G-9** No partial scope creep into deferred items (DS-07/DS-08/occupancy contract) under another heading (BLK-P5-06).
- **G-10** No deletion of availability audit/balance rows from a reservations path (BR-5-042/043).
- **G-11** Ratified Phase 1–4 decisions (§5 register) may not be re-opened; changes require the §3 amendment protocol with new evidence.
- **G-12** Hygiene/dead-code work may not precede behavior-preserving migration.

### 33.3 Additional explicit prohibitions (by domain area)

| Area | Prohibited behavior | Section |
|---|---|---|
| Read contract | A third availability endpoint/contract; server or client deriving available by other than A1; treating occupancy as availability without declared source | §10, §17.4 |
| Restriction semantics | Reading absence as permission for an unproven store; silently preferring a store on conflict; hardcoding "all in scope"; treating `UNRESOLVED` as a resolved restriction | §13.4, §14 |
| Selling permission | Any component outside A1 combining restriction state with quantity; keeping the A3 local overlay after flag-ON | §9.4, §23.4 |
| Isolation | Default-hotel fallback; bare-id availability mutation; aggregating availability across hotels; relying on route exemption instead of scoped authorization; unscoped raw SQL | §22 |
| Frontend | Client-side availability/occupancy math; rendering `UNKNOWN`/`BLOCKED` as a bare number or as permission to book; hardcoding `available: 0`; duplicating contract types; showing legacy/raw signals as authority; assuming literal placeholder copy is final UI (no UI/UX design work here) | §17, §18 |
| API | New availability routes outside the sanctioned set; unvalidated write payloads; message-substring error detection; duplicate route declarations surviving | §21, §27 |
| Write/cutover | Raw restriction writes after DS-04; removing a legacy write leg before its gates; wiring restriction edits to raw writes; flag-ON with the stub bound; silent random operation keys | §23, §24 |
| Delete | Releasing availability during hard-delete; deleting before the deterministic state check; cascade-deleting movements/balances | §26.4 |
| Reconciliation | Any auto-repair, write, or use of reconciliation output in domain decisions; letting legacy win a comparison | §25 |
| Logs/audit | Promoting audit/reconciliation/operational logs to a business authority; conflating log data with availability state | §26.2 |
| Flags/rollout | New availability flags; platform-DB availability flags; code-level flips; unblocking wash; altering the canonicalRead→soak→canonicalWrite sequence | §28 |
| Governance | Executing a code-changing stage without recorded evidence; citing stale baselines as live; partially implementing a deferred item; re-opening a ratified decision outside §3; legacy first, authority later for a new feature (write the new feature against the authority immediately) | §3, §34, §36 |

## 34. Explicitly Deferred Behaviors

### 34.1 Deferral rule

A deferred behavior is **out of Phase 5 scope and may not be partially implemented, simulated, or re-labeled under another heading** (G-9). Deferrals carry explicit authority (cited); re-entry requires the §3 amendment protocol with new evidence.

### 34.2 Deferral register

| Item | Deferred behavior | Authority | What is **not** deferred (the decided remainder) | Re-entry condition |
|---|---|---|---|---|
| **DS-07** outbound channel push | No push contract, no publication of availability to channels/CRS from this phase; no UI/decision use of `channel_availability_log` | P-19, `13_…:65` | Guard rule issued: no non-authority publication exists today and none may be faked (BR-5-030, AC-41) | Explicit channel-push requirement + authority-backed data |
| **DS-08** multi-property | No cross-property queries, rollups, or transfers; every rule in this document is single-hotel (`hotel_id`) | TR-14.3 | Full isolation rule set stands (§22) | Multi-property scope decision with authorization model |
| **DS-06 occupancy analytics contract** | The corporate/analytics occupancy contract is not started; no new dashboard occupancy definition is issued here | P-20 | Availability-labelled display rules are **specified** (BR-5-028/029, AC-42); occupancy must be source-declared and never labelled availability | Analytics contract work |
| **E-2 precedence** | No precedence ranking between conflicting restriction stores | BR-5-003, G-3 | Conflict ⇒ `UNRESOLVED` + `sourceConflicts` is in force now | Evidence of actual conflicts + documented product intent |
| **E-4 CUTOFF store** | `restrictions` (`rate_code='CUTOFF'`) is not migrated into the authority read/write path; `zero_sell_value` semantics undetermined | BR-5-024 | Store flagged as unproven input; excluded from evaluator scope rather than guessed (BR-5-046, AC-40) | Product/domain definition of the store's meaning |
| **E-5 external consumers** | Retirement **schedule** for `/rates/engine/*` awaits the external inventory | BR-5-017 disposition | Dispositions themselves are fixed (§21.4) | Inventory of out-of-repo consumers |
| **E-7 route winner** | Changing the owner of `GET /tax-rates` | BR-5-044 | Single-owner requirement issued; verification duty assigned | Runtime confirmation of serving controller |
| **Legacy `availability` table deletion** | No drop, no migration, no backfill; legacy remains read-only until Phase 11 | P-16, G-1, Phase 4 Deviation C | Read-side retirement proceeds without deletion | Phase 11 legacy retirement scope |
| **Pickup-created assertion shape (DEF-3)** | Pickup-created reservations still bypass the assertion port; port-shape handling is inherited unresolved from Phase 4 | Phase 4 DEF-3 | The rule this affects (BR-5-005 enforcement coverage) is recorded as a dependency in §32 | Phase 4 decision on DEF-3 |
| **Phase 4 open actions** (BLK-1/BLK-2 wash store decisions, cascade-flag deploy configuration, canonical cutover) | Phase 5 does not execute or assume completion of these | `17_…:92-99`, F-24 | Phase 5 requirements reference their state explicitly (§28.5, §32) | Phase 4 closure evidence |
| **Wash/release/attrition activation** | Hourly wash and its metadata authority remain disabled | P-21 + Deviations A/B | Preservation only — Phase 5 changes nothing here | Deviations A and B resolved |
| **Future availability cache** | No cache is built; no stale-by-design behavior | TR-10.3, BR-5-039 | `LIVE_READ` honesty stated (§11.5) | A cache requirement that implements real invalidation |

## 35. Complete Traceability

### 35.1 Findings F-01…F-27 → requirement → decision → specification

| Finding | S3R | Decision / rule anchors | Specification section | Status in this document |
|---|---|---|---|---|
| F-01 unresolved evaluator | S3R-090 | DS-01, BR-5-001…005, 046 | §13, §14, §31 | BLOCKED — BLK-P5-01 (E-1, E-3) |
| F-02 snapshot consumers | S3R-091 | DS-02, BR-5-010…014 | §10, §11, §17, §36 | SPECIFIED |
| F-03 restriction ownership/overlay | S3R-092 | DS-01/DS-04, BR-5-021…023 | §23, §33 | SPECIFIED |
| F-04 dual-write drift | S3R-093 | DS-05, BR-5-025/026 | §24, §25 | SPECIFIED |
| F-05 FO upgrade gate | S3R-094 | DS-03, BR-5-019 | §16.2, §20 | SPECIFIED |
| F-06 unscoped `GET /rates/availability` | S3R-095 | DS-03, BR-5-015/051 | §20, §21.4, §22.7 | SPECIFIED |
| F-07 quick-book/CRS book gate | S3R-096 | DS-03, BR-5-017…019, 050 | §16.1, §21.4 | SPECIFIED (execution gated BLK-P5-03) |
| F-08 canonical restriction routes | S3R-097 | DS-11, DS-04, BR-5-040/041 | §21.5, §17.5 | BLOCKED — DS-04 (wiring) |
| F-09 client-side math/bed types | S3R-098 | DS-02, BR-5-012 | §12.4, §18, §33 | SPECIFIED |
| F-10 quick-book cell mapping | S3R-099 | DS-02, BR-5-011 | §17, §30 AC-22 | SPECIFIED |
| F-11 legacy availability figures | S3R-100 | DS-03, BR-5-045 | §20, §13.3 | SPECIFIED |
| F-12 two screens disagree | S3R-101 | DS-02, BR-5-008 | §9, §30 AC-01 | SPECIFIED |
| F-13 duplicated contract types | S3R-102 | DS-02, BR-5-013 | §11.6, §17.2 | SPECIFIED |
| F-14 cache freshness claims | S3R-103 | BR-5-039 | §11.5, §28.4 | SPECIFIED |
| F-15 channel log non-authoritative | S3R-104 | DS-07, BR-5-030 | §33, §34 | DEFERRED (P-19) + guard SPECIFIED |
| F-16 default-hotel/random identity | S3R-105 | BR-5-027, BR-5-032 | §5.2, §24.5, §22.3 | PRESERVED + SPECIFIED |
| F-17 missing env declarations/zero-harness-skip | S3R-106 | BR-5-033/034 | §36 | SPECIFIED (Stage 4 duties) |
| F-18 no flag documentation | S3R-107 | BR-5-035/036 | §28.2, §36 | SPECIFIED |
| F-19 route-property divergence | S3R-108 | BR-5-047, BR-5-048 | §22.4, §22.5 | SPECIFIED |
| F-20 duplicate `/tax-rates` route | S3R-109 | DS-11, BR-5-044 | §21.7 | BLOCKED — E-7 |
| F-21 dead artifacts | S3R-110 | G-12 | §33, §36 | SPECIFIED (gate) |
| F-22 stale baselines | S3R-111 | BR-5-033, E-8 | §36, §31 | SPECIFIED (evidence at first executed exit) |
| F-23 doc drift | S3R-112 | BR-5-033/036 | §36 | SPECIFIED |
| F-24 Phase 4 actions not done | S3R-113 | §32 carry-overs | §32, §34 | SPECIFIED |
| F-25 duplicate availability routes | S3R-114 | BR-5-044, G-12 | §21.7, §36 | SPECIFIED (hygiene gated) |
| F-26 raw SQL interpolation | S3R-115 | BR-5-021, BR-5-049 | §23.2, §22.6 | SPECIFIED |
| F-27 message-substring error detection | S3R-116 | BR-5-020, E-6 | §27 | SPECIFIED (E-6 informative) |

### 35.2 Decision coverage (DS-01…DS-12)

| DS | Primary sections | Rules | Status carried |
|---|---|---|---|
| DS-01 | §13, §14, §31 | BR-5-001…009, 037, 046 | HARD blocker recorded (BLK-P5-01) |
| DS-02 | §10, §11, §12, §17, §18 | BR-5-010…014 | Released; execution gated by BLK-P5-01 |
| DS-03 | §19, §20, §21 | BR-5-015…020, 050, 051 | CONDITIONAL (BLK-P5-03; E-5 schedule) |
| DS-04 | §23 | BR-5-021…024 | CONDITIONAL (BLK-P5-02; E-4) |
| DS-05 | §24 | BR-5-025…027 | CONDITIONAL (BLK-P5-03) |
| DS-06 | §17.4, §34 | BR-5-028, 029 | Display specified; analytics contract DEFERRED |
| DS-07 | §33, §34 | BR-5-030 | DEFERRED (P-19) + guard SPECIFIED |
| DS-08 | §22, §34 | BR-5-031, 032 | DEFERRED (TR-14.3) + isolation SPECIFIED |
| DS-09 | §36, §31 | BR-5-033, 034 | SPECIFIED (gates exits; E-8 first exit) |
| DS-10 | §28, §36 | BR-5-035…039 | SPECIFIED |
| DS-11 | §21.5, §21.7, §26.4 | BR-5-040…044 | Parts specified; E-7 open; wiring gated DS-04 |
| DS-12 | §13.3, §31 | BR-5-045, 046 | Ownership specified; population BLOCKED (E-1) |

### 35.3 Register completeness

| Class | Register | Total | Specified | Preserved | Blocked | Deferred | Dropped |
|---|---|---|---|---|---|---|---|
| A — business rules | §7.3.1 / §8.2 | 51 | 37 | 10 | 4 (E-1, E-4, E-7, E-1) | 0 | 0 |
| B — blocker requirements | §7.3.2 | 6 | 2 | 0 | 3 | 1 | 0 |
| C — evidence requirements | §7.3.3 | 8 | 1 | 0 | 6 | 1 | 0 |
| D — guardrails | §7.3.4 / §33.2 | 12 | 12 | 0 | 0 | 0 | 0 |
| E — DS entry conditions | §7.3.5 | 12 | 12 | 0 | 0 | 0 | 0 |
| F — finding requirements | §7.3.6 / §35.1 | 27 | 24 | 0 | 3 | 0 | 0 |
| **Discrete total** | | **116** | **88** | **10** | **16** | **2** | **0** |
| G — cross-cutting duty groups | §7.3.7 | 6 groups (members counted above) | | | | | 0 |

Evidence state: 8 evidence items (E-1…E-8) tracked in §31 with named missing evidence; 6 blocker classes in §31/§34; 22 preserved prior-phase decisions (P-1…P-22) in §5; 47 acceptance conditions (AC-01…AC-47); 34 invariants (INV-P5-01…INV-P5-34); 10 migration rules (M-1…M-10) in §19.

## 36. Stage 4 Implementation-Planning Inputs

Stage 4 plans implementation **from this document alone**; it re-derives nothing from the legacy corpus and re-opens no decision.

### 36.1 What may be planned

1. **Behavior-preserving migrations** against the normative boundaries in §19 (M-1…M-10) and §21.4 dispositions — carried as requirements, not re-decided.
2. **The ordered cutover program** exactly as §24.2 states (DS-01 operational → DS-03 dispositions → DS-05 leg removal), with per-step gates: evaluator + E-1 + E-3 before flag-ON; DS-04 write path before affordance wiring; DS-01+DS-03 before any write-leg removal.
3. **Evidence collection work** for E-1…E-8 as explicitly identified evidence gathering (data audits, runtime observations, suite runs) — never as assumed results.
4. **Configuration documentation** duties: seven env `FEATURE_*` flags declared in `compose.yaml`/`.env.example`; `AVAILABILITY_TEST_DATABASE_URL` declared for test/CI (BR-5-034/036).
5. **Hygiene, gated last**: dead-artifact removal (F-21), duplicate-route consolidation (F-25), doc-drift correction (F-23) — only after each behavior-preserving replacement they depend on has landed (G-12).

### 36.2 Evidence required at each code-changing exit

- Full API suite runs recorded with `AVAILABILITY_TEST_DATABASE_URL` set — zero harness-skips for that reason.
- Web typecheck, lint, and test results recorded.
- Baselines reconciled against documented figures (E-8) with deviations explained.
- Flag states reported with their gate evidence (BR-5-035); no code-change side effects.
- Property-scope regression coverage evidenced for any touched availability surface (§22).

### 36.3 Explicit non-inputs (Stage 4 must not plan these)

1. Schema/migration/index/constraint/seed changes (G-1; legacy deletion belongs to Phase 11).
2. New availability flags or platform-DB flag registrations (G-5).
3. Third read contracts, endpoint redesign, or UI/UX design work (G-8; brief scope).
4. Any deferred item from §34, in whole or in part (G-9).
5. Wash activation, canonical write sequence changes, or pickup canonical flags (P-21, Deviations A/B).
6. Task IDs, file-level plans, or schedules derived from anything other than §19/§24/§31 gates — sequencing authority is §24.2.

## 37. Closure Statement

**Authority.** This document is the single authoritative domain specification for Availability Phase 5. It incorporates every Stage 3 requirement/input recorded in Stage 2 §21–§31, preserves ratified Phase 1–4 authority (P-1…P-22) without alteration, and converts Stage 1 findings F-01…F-27 into specified, preserved, blocked, or explicitly deferred requirements — **116 discrete items: 88 SPECIFIED, 10 PRESERVED, 16 BLOCKED (each with a named dependency), 2 DEFERRED WITH EXPLICIT AUTHORITY, 0 silently dropped** (§35.3).

**Contract finalization.** The availability authority model (§9), read contracts (§10–§12), restriction semantics (§13–§15), consumer contracts (§16–§18), migration rules (§19), API contract (§21), isolation contract (§22), write ownership (§23), cutover ordering (§24), reconciliation (§25), audit/logs (§26), error semantics (§27), and flag governance (§28) are finalized as specified, bounded only by the deferred items in §34 and the evidence gates in §31.

**Blockers carried, not resolved here.** The single HARD blocker remains **BLK-P5-01** (DS-01 evaluator + E-1 population audit + E-3 runtime confirmation); conditional blockers BLK-P5-02 (DS-04) and BLK-P5-03 (ordering) constrain execution only; evidence items E-1…E-8 are declared as evidence requirements with exact missing evidence — **none is converted into an implementation task inside this document**.

**Change safety.** Creating this specification made **zero** source-code, schema, migration, test, configuration, dependency, or GitHub changes: only `docs/availability/phase-5/03_FINAL_DOMAIN_SPECIFICATION.md` was added. This stage executed no tests and claims none (AC-43 duties begin at the first code-changing exit).

**Stage boundary.** Stage 3 is complete on delivery of this document. **Stage 4 (implementation planning) has not begun** and must not treat this document as permission to execute any blocked or deferred item.

---

*End of `03_FINAL_DOMAIN_SPECIFICATION.md` — Availability Phase 5, Stage 3 Final Domain Specification.*

# XYLO Availability — Phase 5 — Stage 4: Implementation Plan

> **STRICT PLANNING DOCUMENT. NO IMPLEMENTATION HAS OCCURRED.**
> This document converts the Stage 3 Final Domain Specification into a deterministic implementation plan.
> It creates no code, no schema, no tests, no config, and re-opens no decision.

---

## 1. Document Control

| Field | Value |
|---|---|
| Document | `docs/availability/phase-5/04_IMPLEMENTATION_PLAN.md` |
| Stage | Phase 5 — Stage 4 (Implementation Plan) |
| Workflow position | 1 Forensic Audit → 2 Business Rules/Decisions → 3 Final Domain Specification → **4 Implementation Plan (current)** → 5 Readiness Review → 6 Implementation |
| Primary authority | `03_FINAL_DOMAIN_SPECIFICATION.md` (Stage 3, closed) |
| Supporting authority | `02_BUSINESS_RULES_DECISIONS.md` (Stage 2, closed), `01_FORENSIC_AUDIT.md` (Stage 1, closed) |
| Prior-phase authority | Phase 1–4 corpus (`docs/availability/phase-4/`, `docs/enterprise/*`), ratified, not re-opened |
| Workspace authority | Local workspace `C:\Users\Pro\Desktop\XYLO` only; GitHub is out of scope as authority (no fetch/pull/checkout/restore/history modification) |
| Artifact gate verdict | **STATE B — COMPLETE WITH EMBEDDED COVERAGE** (§4) → plan authorized |
| Code/schema/test/config changes made producing this document | **0** |
| Files created/modified by this stage | this file only |
| Implementation status | **NOT STARTED.** Stage 5 (Readiness Review) NOT started. Stage 6 (Implementation) NOT started. |

---

## 2. Stage 4 Scope

**In scope:**

1. Artifact Completeness / Phase 4 → Phase 5 parity validation (§4) before any planning output.
2. Stage 3 validation — registers, decisions, blockers, evidence, invariants, acceptance conditions, migration rules (§5). Validation only; no redesign.
3. Conversion of the Stage 3 specification into: workstreams, deterministic tasks, dependency-aware execution order, parallelization, evidence/blocker gates, test and static-analysis strategy, traceability, rollback boundaries, and Stage 6 quality gates.
4. Explicit treatment of every Stage 3 requirement/input as SPECIFIED / PRESERVED / BLOCKED / DEFERRED (§31).

**Out of scope (hard):** any implementation; any second Requirements stage; reopening Stage 1–3 decisions; UI/UX design; schema/migration changes; flag changes; test execution; GitHub operations; creating any file other than this one.

**Task-design rule:** every task states WHAT/WHY/WHICH RULE/WHICH EVIDENCE/ORDER/VERIFICATION/COMPLETION/ROLLBACK. **No task may require an implementation agent to invent a business rule** — where a rule or evidence is missing, the task is blocked with a named gate, never "decided during implementation."

---

## 3. Authority and Inputs

### 3.1 Authority tiers (carried from FDS §3; unchanged)

1. Stage 3 Final Domain Specification (this phase's domain authority).
2. Stage 2 decision register (rules canonical text: Stage 2 §20).
3. Stage 1 forensic evidence (findings, inventories, evidence index).
4. Ratified Phase 1–4 documents (P-1…P-22, TR-*, contract extracts, Phase 4 plan/RR records).
5. Local source/schema/test evidence (file:line citations).

A lower tier never overrides a higher tier; conflicts are documented, never silently resolved (FDS §3.4).

### 3.2 Approved inputs used

- `docs/availability/phase-5/01_FORENSIC_AUDIT.md` — 16 sections; F-01…F-27; inventories (consumer, legacy, frontend, API, flags, tests); evidence index §15; migration boundaries §14 (B-1…B-8).
- `docs/availability/phase-5/02_BUSINESS_RULES_DECISIONS.md` — 31 sections; DS-01…DS-12 with option matrices; canonical rules BR-5-001…051 (§20); blockers §21; dependency map §22; traceability §23; isolation §25; frontend §26; API §27; carry-overs §29; evidence/guardrails §30; closure §31.
- `docs/availability/phase-5/03_FINAL_DOMAIN_SPECIFICATION.md` — 37 sections; the domain contract (authorities §9–§29), registers (§7, §8, §35), Stage 4 inputs (§36), prohibitions (§33), deferrals (§34).
- Phase 1–4 authority: `phase-4/10_DECISION_RESOLUTION.md`, `11_TARGET_BUSINESS_RULES.md`, `13_FINAL_DOMAIN_SPECIFICATION.md`, `14_IMPLEMENTATION_PLAN.md`, `15_READINESS_REVIEW.md`, `16_RR_DECISIONS.md`, `17_PHASE4_READINESS_REVIEW.md` (deviations A–D, baselines, flag states); `docs/enterprise/availability-phase1-2-contract-extract.md`; `…-phase3-domain-specification.md`.

### 3.3 What this plan may and may not derive

May derive: ordering, task decomposition, file targets, verification strategy, gate placement. May NOT derive: new business rules, precedence rankings, deferred-scope work, flag semantics, endpoint shapes beyond FDS §21, or any change to a closed decision (G-11).

---

## 4. Phase 4 → Phase 5 Artifact Parity Validation

### 4.1 Method

Phase 4 shipped 17 artifacts across ~520KB. This gate compares each forensic/architectural layer Phase 4 established against the three Phase 5 documents (323KB combined), asking one question per row: **does implementation planning have the evidence this layer existed to provide?** No document is split for parity's sake (brief §5: parity of coverage, not file count).

### 4.2 Parity matrix

| # | Phase 4 Artifact / Evidence Layer | Phase 5 Coverage | Location | Completeness | Action |
|---|---|---|---|---|---|
| 1 | Forensic Audit | Full audit: summary, scale, explicit non-goals, 16 sections, F-01…F-27 findings, evidence index, UNVERIFIED declarations | `01_FORENSIC_AUDIT.md` §1–§16 (78KB vs P4's 11.9KB) | **COMPLETE** | Keep |
| 2 | Current Architecture | Five coexisting worlds; direction-of-flow; legacy counter writer; A3 second engine; raw-SQL inventory; ad-hoc computations; consumer wiring status | `01` §2 (2.1–2.2), §4.1–4.4, §6.1, §7.1 | **COMPLETE BUT EMBEDDED** | Keep (Phase 4's standalone `02_CURRENT_ARCHITECTURE.md` was GBA-command-flow scoped; Phase 5's availability topology lives in audit §2/§4) |
| 3 | Data Model Audit | Legacy `availability` table (schema refs), raw SQL against restriction tables, 11-store ownership/scope tables with `schema.prisma` + source-line evidence, assertion-table ownership | `01` §4.2–4.3; `02` §19.1; `03` §13.2–13.3, §26.3 | **COMPLETE BUT EMBEDDED** (schema/column evidence at cited file:line); **population data = declared gap E-1** (hard-gated, §10) | Keep; E-1 collected by T5-05 before evaluator scope closes |
| 4 | Availability Integration Matrix | Direction of integration; A1-vs-A3-vs-legacy divergence per formula dimension; every availability-adjacent endpoint classified; frontend consumer map (which screen hits which computation) | `01` §2.2, §7.2, §9, §5.1 | **COMPLETE** | Keep |
| 5 | Current Business Rules | As-implemented behavior matrix (sellable formula, block/allotment eligibility, restriction behavior, consumption, checkout, OOO, overbooking, fail-closed, pickup, persistence, tenancy) + Phase 1–4 compliance matrix | `01` §8, §9 (12-row behavior matrix), §13; `02` §6 | **COMPLETE BUT EMBEDDED** | Keep (Phase 4's `05_…` was GBA rules; Phase 5's current-state rules are availability-specific and live in audit §9 with per-row evidence) |
| 6 | Currency / Transaction analysis | Currency: no money/pricing changes in Phase 5 (rates pricing retained — FDS §21.4). Transaction: assertion-in-transaction semantics, dual-write single-transaction state, conflict/idempotency semantics, delete-before-mutation ordering | Currency: N/A (FDS §2.2 out of scope). Transaction: `01` §6.1; `03` §16.1, §24.1, §24.5, §27, §26.4 | **NOT APPLICABLE (currency) / COMPLETE BUT EMBEDDED (transaction)** | Keep; Phase 4's `06_…` covered GBA read-modify-write hazards — those surfaces are outside Phase 5's change set; Phase 5 introduces no new concurrency mechanism (no schema, no new locks) |
| 7 | Legacy Dependency analysis | Legacy writer location, table access, raw SQL, ad-hoc computations, dead DI, orphan adapters, endpoint dispositions, migration boundaries | `01` §4, §14 (B-1…B-8), §15; `03` §19, §20, §21.4 | **COMPLETE** | Keep |
| 8 | Open Decisions | 12 Stage-2 decision items raised by Stage 1, resolved with option matrices; residual openness isolated as E-1…E-8 evidence items | `01` §12; `02` §7–§19; `03` §31 | **COMPLETE** | Keep |
| 9 | Audit Summary | Headline, scale, conclusion, boundary statement | `01` §1, §16 | **COMPLETE** | Keep |
| 10 | Decision Resolution | Method, option matrices, per-decision analysis (12 decisions + sub-decisions), evidence-based resolution | `02` §5 method, §7–§19 (option matrices), §31 closure | **COMPLETE** | Keep |
| 11 | Target Business Rules | 51 canonical testable rules with source/input/condition/output/prohibited behavior, grouped by DS topic | `02` §20 (canonical text) + `03` §9–§29 (contract form) | **COMPLETE** | Keep |
| 12 | Decision State | Status, blocker class, rules, entry condition per decision; counts; closure | `02` §31.1–31.3; `03` §6.1, §35.2 | **COMPLETE** | Keep |
| 13 | Consumer Inventory | Complete inventory of all availability consumers (backend, frontend, integrations, jobs, reporting) | `01` §3 (+ §5, §6 detail) | **COMPLETE** | Keep |
| 14 | Backend Integration Inventory | Reservations write paths (per-operation authority/legacy verdict), Front Office per-operation, Group/Allotment, A3, Channels, Events, analytics, dead DI | `01` §6.1–6.4 | **COMPLETE** | Keep |
| 15 | Frontend Integration Inventory | 17 display surfaces with file:line, 14 API client calls, 11 client-math formulas, 5 shadow stores, dead/broken artifacts, flags, admin/mobile, tests | `01` §5.1–5.8 | **COMPLETE** | Keep |
| 16 | API Contract Inventory | Authority surface + response schema, classification of every availability-adjacent endpoint, DTO/contract gaps, gateway | `01` §7.1–7.5; `03` §21 | **COMPLETE** | Keep |
| 17 | Database/Data Evidence | Schema-cited legacy table, raw SQL inventory, store column evidence at source lines, assertion-table ownership, grep evidence (E6/E10), schema-change prohibition | `01` §4.2–4.3, §15.2–15.3; `03` §13.2–13.3, §23.2, §36.3 | **COMPLETE BUT EMBEDDED**; population facts = E-1 (declared, never assumed) | Keep; E-1 (T5-05) is the data gate; G-1 forbids schema work |
| 18 | Dependency Graph | Decision dependency map, hard ordering (DS-01→DS-03→DS-05), migration boundaries, sub-ordering table, Stage 4 planning inputs | `02` §22; `03` §24.2, §36.1; `01` §14 | **COMPLETE** | Keep; task-level graph produced in §29/§30 of this plan |
| 19 | Migration/Cutover Evidence | 10 migration rules, endpoint-by-endpoint dispositions, dual-write cutover conditions, rollback boundaries, flag rollout sequence | `03` §19, §20, §21.4, §24, §28.5 | **COMPLETE** | Keep |
| 20 | Risk/Blocker Register | 11 risks + dependencies; 6 blocker classes; evidence gates; deferral register; prohibited behaviors | `01` §11; `02` §21, §30; `03` §31, §33, §34 | **COMPLETE** | Keep |

Phase 4 artifacts with **no Phase 5 counterpart by design**: `08_OPEN_DECISIONS` (resolved into `02` §7–§19), `14_IMPLEMENTATION_PLAN` (produced now), `15/16/17` Readiness Review family (Stage 5/6 products, §35).

### 4.3 Verdict

**STATE B — COMPLETE WITH EMBEDDED COVERAGE.** All 20 required layers exist with implementation-usable depth inside the three Phase 5 documents; the single declared data gap (population facts, E-1) is a Stage 3-registered hard evidence gate with its own gate task — not a missing artifact. No document is split for parity; no missing evidence is assumed. **Proceeding to implementation planning.**

---

## 5. Stage 3 Validation (no redesign)

### 5.1 Register verification (executed against `03_FINAL_DOMAIN_SPECIFICATION.md`)

| Register | Expected | Verified | Result |
|---|---|---|---|
| Decision items | DS-01…DS-12 | 12 unique, all referenced with status in §6.1/§35.2 | ✅ |
| Business rules | BR-5-001…051 | 51 unique; all 51 mapped in §8.2 with section + status | ✅ |
| Blockers | BLK-P5-01…06 | 6 unique; classes HARD/CONDITIONAL×2/NON-BLOCKING×2/DEFERRED in §7.3.2 | ✅ |
| Evidence items | E-1…E-8 | 8 unique; each with missing-evidence statement, dependent sections, failure behavior in §31 | ✅ |
| Guardrails | G-1…G-12 | 12 unique, reproduced verbatim in §33.2, invariants cross-linked §29 | ✅ |
| Acceptance conditions | AC-01…AC-47 | 47 unique, zero undefined references, all referenced | ✅ |
| Invariants | INV-P5-01…34 | 34 unique | ✅ |
| Migration rules | M-1…M-10 | 10 unique in §19 | ✅ |
| S3R inputs | S3R-001…116 + S3R-G1…G6 | 116 discrete items (51+6+8+12+12+27) + 6 duty groups; status totals 88 SPECIFIED / 10 PRESERVED / 16 BLOCKED / 2 DEFERRED / 0 dropped (§7.4, §35.3) | ✅ |
| Findings | F-01…F-27 | 27 dispositioned, full chain in §35.1 | ✅ |
| Preserved decisions | P-1…P-22 | register in §5.1 of FDS, cited throughout | ✅ |

### 5.2 Explicit Stage 4 inputs (FDS §36) carried into this plan

- §36.1(1) behavior-preserving migrations against M-1…M-10 + §21.4 dispositions → workstreams §16/§19/§20.
- §36.1(2) ordered cutover program with per-step gates → §18, §29.
- §36.1(3) evidence collection for E-1…E-8 as evidence tasks (never assumed results) → §10.
- §36.1(4) configuration documentation duties (7 flags; `AVAILABILITY_TEST_DATABASE_URL`) → §22, tasks T5-54.
- §36.1(5) hygiene gated last (F-21/F-25/F-23 after replacements) → T5-42/43, G-12.
- §36.2 exit evidence: full API suite with DB env (zero harness-skips), web typecheck/lint/test, baseline reconciliation (E-8), flag states with gate evidence, property-scope regression → §34.
- §36.3 non-inputs honored: no schema work, no new flags, no third contract/UI design, no deferred items, no wash/sequence changes, no task IDs derived from anything but §19/§24/§31 gates → §36.

### 5.3 Validation outcome

Stage 3 stands as written. **No decision reopened; no requirement dropped; no evidence replaced by assumption.** Genuine contradictions: none found (the only cross-document tensions are the declared E-items and Phase 4 carry-overs, all documented, not silently resolved).

---

---

## 6. Current Implementation Baseline (verified, read-only)

**Repository state:** HEAD `c854f79` ("Phase 0 — Enterprise Platform Foundation complete"), 4 commits; `git status --short` = **776 dirty paths** (pre-existing Phase 4 Stage-6 delta of 766 uncommitted files + others; standing rule: never committed). Stages 1–4 of Phase 5 added only untracked `docs/availability/phase-5/` files — dirty count unchanged at 776.

**Authority chain state (F-01):** `availability.module.ts:26` binds `RESTRICTION_EVALUATOR → UnresolvedRestrictionAdapter`, which hardcodes `status: 'UNRESOLVED'` and pushes 7 sources into `unresolvedSources`. Snapshot propagation: `availability-snapshot.service.ts:104-107` ⇒ `sellableAvailable: 0` + `bookingEligibility: 'UNKNOWN'`. Assertion chain: `unassertableReason` (`availability-assertion.service.ts:881-893`) ⇒ `UNRESOLVED_CAPACITY` ⇒ `REJECTED` ⇒ `AVAILABILITY_ASSERTION_REJECTED` (409) with transaction rollback (`reservation-availability-wiring.ts`). **No end-to-end test wires the production binding** (specs inject resolved fixtures; harness hardcodes `ELIGIBLE`).

**Consumer state:** zero consumers of `/availability/snapshot` and `/availability/reconciliation` (grep evidence 01 §15.3-E1). Frontend entirely on legacy/A3 endpoints. Matrix dual-mode: flag read at `availability-sales.controller.ts:275`, overlay `:540`, interim label `:612`. Dual-write live at `reservation.repository.ts:621/:625`. FO upgrade gate MIX at `upgrade-room.handler.ts:69/:85`. A3 raw restriction writes at `availability-sales.controller.ts:676-707, 825-885`. Broken frontend routes: `reservation.api.ts:405` (`/availability/matrix/logs` ⇒ 404), `:431` (`/activities/availability/interval-update` ⇒ 404, payload also wrong).

**Test baselines (Phase 4 T-70 battery — preserved, never redefined):**

| Baseline | Value |
|---|---|
| API full suite (jest, DB env) | 178 suites / 1470 pass / **1 skip** / **6 fail** |
| Known API baseline failures (do-not-fix) | 2 suites / 6 tests: `front-office/check-in/…/queries.handler.spec.ts` (3) + `reservations/…/reservation-integration.spec.ts` (3) |
| Sole `it.skip` | `t45-execute-cut-off-wash.spec.ts:293` (Deviation A marker) |
| GBA battery | 79 suites / 766 pass / 1 skip / 0 fail |
| Web baseline failure | 1 suite / 10 tests: `features/reservations/model/__tests__/reservation-actions.test.ts` |
| Web typecheck / lint | 0 / exit 0 |
| Test inventory | 178 `*.spec.ts` (api), 48 `*.postgres.spec.ts`, 5 web test files; web has **zero** availability coverage |
| Prisma validate | main + inventory both valid (2 pre-existing SetNull warnings) |

**Flag state (Phase 4 §6 — preserved):** `gba.consumers.cascade` **MUST be ON at deploy**; `gba.wash.schedulerEnabled` BLOCKED on Deviations A+B; `canonicalRead` → soak → `canonicalWrite` READY sequence untouched; `gba.a3.authoritative` per its own BLK-P5-01 gate; all default OFF via `ConfigService.getFeatureFlag` (`config.service.ts:63-65`); `FEATURE_*` appears **nowhere** in `.env`/`.env.example`/`compose.yaml` (grep 01 §15.3-E3).

---

## 7. Implementation Principles

1. **Specification is law.** FDS §9–§33 defines behavior; this plan only sequences and verifies it. Any discovered contradiction is raised, not resolved locally (FDS §3.4, G-11).
2. **Fail-closed default.** Unresolved ⇒ 0 floors + `UNKNOWN` + rejected assertion; never suppress `unresolvedSources`; never treat `UNKNOWN` as `ELIGIBLE` (G-2).
3. **Evidence over assumption.** E-1…E-8 are gates with recorded evidence; missing evidence blocks, never guesses (BR-5-046, TR-10.4).
4. **Behavior-preserving first; hygiene last.** Replacement ships before removal (M-10, G-12).
5. **Ordering is a gate, not a preference.** DS-01 operational → DS-03 dispositions → DS-05 leg removal; flag flips are operational actions with recorded evidence, never code side effects (FDS §24.2, G-5).
6. **Single authority, single combination.** No second sellable number; no client-side availability math; no third contract (G-8).
7. **Isolation by construction.** Every read/write/job/log carries `hotel_id`; route property is authoritative; `'default'` rejected (G-7).
8. **Rollback = previous behavior.** Every switch is a flag or a redeploy; no schema rollback is ever needed because there is no schema change (G-1, FDS §24.7).
9. **Deterministic tasks.** Precondition, dependency, action, test, verification, completion, and rollback are stated per task; blocked tasks carry a named unblock condition.
10. **Preserve baselines.** Known failures and the sole skip stay as documented; no silent redefinition (DS-09).

---

## 8. Dependency Model

### 8.1 Gate classes

| Class | Meaning | Example |
|---|---|---|
| **EVIDENCE gate** | task cannot start (or cannot be trusted) until named evidence E-x recorded | T5-07 needs E-1 |
| **BLOCKER gate** | BLK-P5-0x open ⇒ affected work held | BLK-P5-01 ⇒ T5-09 |
| **ORDER gate** | Stage 3 §24.2 sequencing must hold | DS-01 → DS-03 → DS-05 |
| **CUTOVER gate** | activation action requiring recorded gate evidence | flag ON (T5-56) |
| **HYGIENE gate** | G-12: replacement must exist first | T5-42 |
| **EXIT gate** | stage-exit evidence battery (DS-09/§36.2) | T5-76 |

### 8.2 Hard dependency chains (derived from FDS §24.2, §7.3, §31)

```
CHAIN-1 (authority enablement — serial, critical path):
  T5-05 (E-1 population audit) ─┐
  T5-06 (E-3 runtime confirm) ─┼─► T5-07 (evaluator build) ► T5-08 (semantics suite)
                               └──────────────► T5-09 (DI binding swap)
  T5-09 ► T5-10 (parity soak) ─► T5-56 (gba.a3.authoritative ON gate)
  Gates: BLK-P5-01 opens only when T5-07+T5-08+T5-05+T5-06 all green.

CHAIN-2 (frontend cutover — after CHAIN-1):
  T5-29 (hook/types, ungated) ► T5-30 (AvailabilityPage cutover) ► T5-31 (quick-book) ► T5-32 (rates tab)
  T5-40 (overlay removal, flag-ON window) rides T5-56.

CHAIN-3 (booking-path retirement — after CHAIN-1 + read dispositions):
  T5-13 (FO upgrade gate) , T5-14 (CRS quote/book signal) , T5-22…T5-25 (engine retirements)
  T5-24 (book retire) specifically: DS-01 operational + T5-14 + T5-31 in place.

CHAIN-4 (write convergence — last):
  T5-49 (status-quo pin) ► [CHAIN-1 closed] ► [CHAIN-3 read/write gates closed] ► T5-50 (remove :625 leg)
  T5-53 ordering checklist runs at every cutover step.

CHAIN-5 (restriction ownership — parallel to CHAIN-2, gates UI):
  T5-45 (authority write path) ► T5-46 (A3 handlers call it) ► T5-48 (affordances wiring, BLK-P5-02)
  T5-39 (raw display reads removal) after T5-45/46 read side.
  T5-47 (CUTOFF exclusion) independent guard.

CHAIN-6 (evidence-only gates): T5-87 (E-4), T5-88 (E-5 → schedules T5-22…25), T5-89 (E-6, informative),
  T5-90 (E-7 → gates T5-27), T5-76 (E-8 at first executed exit).

CHAIN-7 (hygiene — after their replacements): T5-42, T5-43.
```

### 8.3 What may never precede what (normative)

1. Nothing in CHAIN-2/3/4 before CHAIN-1 evidence closes (BLK-P5-01).
2. T5-50 before both DS-01 operational and DS-03 dispositions landed (BLK-P5-03).
3. T5-48 before T5-45 (BLK-P5-02); until then affordances visibly disabled — never silent 404 (AC-45).
4. T5-42/43 before no one — they are last (G-12).
5. T5-56 before flag-ON claims; flag never flips inside a code change (G-5).

---

## 9. Blocker Register

| Blocker | Class | Reason | Affected work (tasks) | Prerequisite | Unblock condition | Verification |
|---|---|---|---|---|---|---|
| **BLK-P5-01** | HARD | Evaluator unshipped (stub ⇒ UNRESOLVED ⇒ 0/UNKNOWN ⇒ 409); E-1 population facts unknown; E-3 runtime unconfirmed | T5-07/09/10, T5-30/31/32, T5-13/14, T5-24, T5-40, T5-50, T5-56 | T5-05, T5-06, T5-07, T5-08 | evaluator bound + E-1 recorded + E-3 recorded + parity soak (T5-10) green | recorded evidence files + suite green |
| **BLK-P5-02** | CONDITIONAL | Restriction-edit UI wiring would cement raw legacy writes | T5-48, T5-33 (affordance portion) | T5-45 (authority write path) | write path landed; affordances wire to it or stay visibly disabled | AC-45 test |
| **BLK-P5-03** | CONDITIONAL | Cutover ordering: authority first, then readers disposed, then writers removed | T5-24, T5-50 (and retirement timing of T5-22…25) | CHAIN-1 + read dispositions | §24.2 order satisfied step-by-step | T5-53 checklist per step |
| **BLK-P5-04** | NON-BLOCKING | Evidence policy gates stage exits, not decisions | T5-76 and every exit battery | none | each exit records executed suite evidence (§34) | exit gate report |
| **BLK-P5-05** | NON-BLOCKING | Flag documentation/governance | T5-54, T5-55, T5-56 | none | flags documented; zero code-level flips | T5-55 test + doc diff |
| **BLK-P5-06** | DEFERRED | DS-07 push, DS-08 multi-property, DS-06 analytics contract | **no tasks** (G-9) | n/a | explicit future authority | scope scan (T5-N/A — §36 non-inputs) |

**No blocker may be silently bypassed or closed.** Closure = evidence recorded + dependent tests green + note in the Stage 5 readiness inputs (§35).

---

## 10. Evidence Gate Register (E-1…E-8)

| ID | Required evidence | Source | Gate task | Dependent tasks | Status | Validation | Failure behavior |
|---|---|---|---|---|---|---|---|
| **E-1** | Per-hotel row counts, recency, writer identification for `room_inventory`, `out_of_order`, `out_of_service`, `rate_restrictions` + six A3 restriction tables | local DB (read-only queries) + repo writer search | T5-05 | T5-07 (scope), T5-09, CHAIN-2/3 | **OPEN — hard** | evidence record lists store → count/recency/writer-or-UNKNOWN per hotel | store excluded or contribution flagged per BR-5-046; BLK-P5-01 stays open; **no default assumptions** |
| **E-2** | Precedence intent (only if actual store conflicts exist) | product/domain documentation | collected inside T5-05 (conflict detection) | none while default holds | **DEFERRED (default = conflict ⇒ `UNRESOLVED`)** | conflicts found ⇒ STOP and request product decision | conflict behavior stays fail-closed (BR-5-003); no ranking invented |
| **E-3** | Runtime confirmation that production DI chain behaves as static analysis predicts (stub ⇒ UNRESOLVED ⇒ 409 + rollback) | executed observation in a safe environment | T5-06 | T5-09, T5-56 | **OPEN — hard** | recorded request/response/rollback evidence | BLK-P5-01 stays open; flag stays OFF |
| **E-4** | Business meaning of `restrictions` (`rate_code='CUTOFF'`) and `zero_sell_value` | product/domain definition | T5-87 | T5-47 scope only | **OPEN — non-blocking** | documented definition | store stays excluded from authority path (BR-5-024); no migration attempted |
| **E-5** | Inventory of out-of-repo consumers of `/rates/engine/*` | external-consumer inventory (ops query) | T5-88 | **timing only** of T5-22/23/24/25 | **OPEN — non-blocking for disposition** | inventory recorded | dispositions unchanged; retirement schedule waits; in-repo consumers proceed |
| **E-6** | Live behavior of OTA overbooking branch (`webhook.service.ts:104`) | runtime check | T5-89 | none (rule already issued) | **OPEN — informative** | observation recorded | BR-5-020 rule stands regardless |
| **E-7** | Runtime winner of duplicate `GET /tax-rates` | runtime request trace | T5-90 | T5-27 | **OPEN — non-blocking** | which controller serves observed | owner not changed until observed; no aliasing attempted |
| **E-8** | Reconciled executed baselines | full test runs with DB env | T5-76 | every code-changing exit | **OPEN — first executed exit** | numbers vs §6 baselines explained | exit blocked until reconciled; baselines never silently redefined |

Evidence items are **evidence tasks, never coding tasks** (FDS §31). E-1/E-3 are the hard path for BLK-P5-01 (T5-05/T5-06 defined in §13). Remaining evidence tasks:

**T5-87 — E-4 CUTOFF meaning evidence** `[evidence] [GATE E-4]`
- **Why/Authority:** BR-5-024; FDS §31 E-4. **Action:** request/locate product-domain definition of `restrictions (rate_code='CUTOFF')` and `zero_sell_value` (repo docs, ops records, domain owner) — read-only. **Validation:** documented definition or documented "meaning unknown". **Failure:** meaning unknown ⇒ store remains excluded forever in this phase (AC-31), no migration, evaluator scope unchanged (FDS amendment would be required to change scope). **Completion:** evidence note filed; T5-47 guard remains.

**T5-88 — E-5 external consumer inventory** `[evidence] [GATE E-5 (timing)]`
- **Why/Authority:** BR-5-017/018 schedule dependency; FDS §31 E-5. **Action:** ops-side inventory of out-of-repo consumers hitting `/rates/engine/*` (gateway logs/config review — local evidence only; no GitHub). **Validation:** inventory recorded (none found ⇒ schedule free). **Failure:** inventory unknown ⇒ in-repo retirements proceed, external-facing retirement waits (disposition unchanged — G-4 keeps the rule). **Completion:** note filed; T5-22…25 timings annotated.

**T5-89 — E-6 OTA overbooking branch live check** `[evidence] [GATE E-6 (informative)]`
- **Why/Authority:** BR-5-020 evidence; FDS §31 E-6. **Action:** observe `webhook.service.ts:104` branch behavior (sandbox). **Validation:** observation recorded. **Failure:** none — the code-detection rule stands regardless. **Completion:** note filed.

**T5-90 — E-7 `/tax-rates` runtime winner** `[evidence] [GATE E-7]`
- **Why/Authority:** BR-5-044; FDS §21.7. **Action:** runtime request trace showing which controller serves `GET /tax-rates` (registration order says `banquet-refs.controller.ts:45` first — runtime must confirm). **Validation:** observed responder recorded. **Failure:** unconfirmed ⇒ T5-27 waits (route untouched — current behavior preserved). **Completion:** note filed; feeds T5-27.

---

## 11. Workstream Architecture

| WS | Workstream (prompt §10) | Spec anchor | Tasks | Nature |
|---|---|---|---|---|
| **A** | Availability Authority (ownership, snapshot, matrix sourcing, fail-closed, UNKNOWN, UNRESOLVED_CAPACITY) | FDS §9–§15 | T5-01…04 | mostly `verify` + `create` (types) |
| **B** | Restriction / Inventory (DS-01, E-1/E-3, room inventory, OOO/OOS, rate restrictions, population evidence, blocker handling) | FDS §13, §14, §31 | T5-05…11 | `evidence` + `create` + `modify` (gated) |
| **C** | Other Consumers (reservations, FO, GBA/blocks/allotments, upgrades, rates, admin, workers, integrations, operational, reporting) | FDS §16 | T5-12…18 | `verify` + `modify` (gated) |
| **D** | API (snapshot, matrix, logs, legacy routes, payload migration, retirement, compatibility, errors, isolation) | FDS §21 | T5-19…28 | `test` + `modify` + `verify` |
| **E** | Frontend (screens, matrix, logs, reservations, FD, upgrades, GBA, allotments, hooks, React Query, Zustand, clients, forms, UNKNOWN handling, client-math removal) | FDS §17, §18 | T5-29…38 | `create` + `modify` + `test` |
| **F** | Legacy Retirement (per-source: location, behavior, classification, replacement, condition, verification) | FDS §19, §20 | T5-39…44 | `modify` + `verify` + gated `hygiene` |
| **G** | Restriction Write Ownership (DS-04: writers, canonical writer, cutover, reconciliation, auditability, retirement) | FDS §23 | T5-45…48 | `create` + `modify` + `verify` |
| **H** | Dual Write / Cutover (DS-05 exact ordering, write identity) | FDS §24 | T5-49…53 | `verify` + gated `modify` |
| **I** | Feature Flags (DS-10 single mechanism) | FDS §28 | T5-54…56 | `config` + `test` + `ops` |
| **J** | Logs / Audit / Reconciliation (incl. DS-11 delete-reject-when-availability-state) | FDS §25, §26 | T5-57…61 | `test` + `verify` + `modify` |
| **K** | Database / Schema (zero-change verification) | G-1, FDS §23.3 | T5-62…63 | `gate` + `verify` |
| **L** | Property Isolation (hotel_id, propertyId, tenant context, reads/writes/jobs/logs/recon/API/frontend) | FDS §22 | T5-64…68 | `test` + `verify` |
| **M** | Testing (unit/integration/DB/API/frontend/regression/GBA/isolation/concurrency) | FDS §36.2, DS-09 | T5-69…76 | `test` + `evidence` |
| **N** | Static Analysis (10 scan families) | FDS §33, §26/§18 prohibitions | T5-77…86 | `verify` (scan gates) |

**Task total: 90** (T5-01…T5-90). Sections 12–26 define every task with authority, preconditions, files, tests, verification, acceptance, risk, rollback, and completion condition.

---

## 12. Availability Authority (WS-A)

The authority itself already exists (Phase 1–3 contract, unchanged). WS-A does not rebuild it: it **pins** its contract with tests, consolidates shared types, and locks state distinctness so downstream migrations cannot drift.

**T5-01 — Snapshot contract conformance suite** `[verify] [WS-A]`
- **Why/Authority:** FDS §11 (response semantics, floors, fail-closed, freshness); BR-5-005/006; INV-P5-04/07; AC-03/04/05/06. The authority's observable behavior must be pinned before any consumer migrates onto it.
- **Scope/Files:** `apps/api/src/modules/availability/application/services/availability-snapshot.service.ts`, `domain/policies/snapshot-calculator.ts`, new/extended specs under `__tests__`.
- **Preconditions/Dependencies:** none (parallel-safe opener).
- **Action:** add tests asserting: `max(0,·)` floors on every quantity; `sellableCapacity = min(capacityWithOverbooking, max(0, sellLimit))` tighten-only; unresolved ⇒ `physicalAvailable=0`, `sellableAvailable=0`, `bookingEligibility='UNKNOWN'`, consumption `null`, `unresolvedSources` non-empty; resolved-zero ⇒ `ELIGIBLE`; blocked ⇒ `BLOCKED` + `sellableAvailable=0`; `overbookingUsed = max(0, consumption − physicalCapacity)` reported; `freshness='LIVE_READ'`.
- **Tests/Verification:** unit suite green (`cd apps/api && pnpm test -- snapshot`).
- **Acceptance:** AC-03/04/05/06 green; INV-P5-04/07/33 pinned.
- **Risk/Rollback:** test-only; revert commit if a pin contradicts ratified formula (then STOP — decision conflict, FDS §3.4).
- **Completion:** suite green and cited in the exit battery.

**T5-02 — Single shared contract types (snapshot + projection)** `[create] [WS-A]`
- **Why/Authority:** BR-5-013; FDS §11.6; AC-21; audit §7.3 gap ("no shared TypeScript type exists").
- **Scope/Files:** `packages/shared` (new `availability-contract.ts`: snapshot response type, reconciliation type, matrix projection cell type incl. additive `bookingEligibility`/`unresolvedSources`/`label`); consumers to import: `apps/api` availability DTOs/controllers, `apps/web` API layer.
- **Preconditions/Dependencies:** none.
- **Action:** define types exactly per FDS §11.3/§11.4/§12.1/§12.5 and audit §7.1 response schema; export; re-point API DTO typing and (later, per task) frontend declarations to these types. No payload shape changes — types mirror the existing contract.
- **Tests/Verification:** typecheck both apps; a compile-time test asserting API response assignable to shared type.
- **Acceptance:** AC-21; single declaration (static scan T5-82 finds zero duplicate availability payload declarations).
- **Risk/Rollback:** additive only; revert on type mismatch with live shape (shape wins — no contract change implied).
- **Completion:** `pnpm typecheck` green in api + web; shared module exported.

**T5-03 — State-model distinctness pins** `[verify] [WS-A]`
- **Why/Authority:** FDS §15 state space A–G; INV-P5-05/34; AC-05/06/19.
- **Scope/Files:** snapshot service specs + evaluator outcome specs (T5-08) + matrix projection test (T5-20).
- **Preconditions/Dependencies:** pairs with T5-01/T5-08.
- **Action:** assert actual-zero ≠ blocked ≠ unknown/unresolved ≠ projection label ≠ error — each rendered with its own state; no path collapses them into one number.
- **Tests/Verification:** unit matrix over the 5 distinctions.
- **Acceptance:** AC-05/06/19 behaviorally pinned.
- **Risk/Rollback:** test-only.
- **Completion:** suite green.

**T5-04 — Authority rejection determinism pins** `[verify] [WS-A]`
- **Why/Authority:** FDS §27 row 1–4; Phase 2 contract; INV-P5-06; AC-12.
- **Scope/Files:** `availability-assertion.service.ts` (`:865-905`), `reservation-availability-wiring.ts`, error mapping.
- **Preconditions/Dependencies:** none.
- **Action:** assert: `UNKNOWN` ⇒ `REJECTED`/`UNRESOLVED_CAPACITY` persisted, `AVAILABILITY_ASSERTION_REJECTED` (409) with `rejection.code`, transaction rolled back, **no partial state**; blocked ⇒ `CAPACITY_BLOCKED`; zero ⇒ `INSUFFICIENT_CAPACITY`; consumers can match codes only.
- **Tests/Verification:** assertion unit + integration (harness) tests.
- **Acceptance:** AC-12; INV-P5-06; sets up AC-27.
- **Risk/Rollback:** test-only.
- **Completion:** suite green with DB env recorded (harness).

---

## 13. Restriction Evaluator (WS-B, DS-01)

**The hard blocker lives here.** BLK-P5-01 = evaluator + E-1 + E-3. T5-05/T5-06 are evidence tasks; T5-07/T5-08 build the requirement; T5-09 swaps the binding only when evidence is recorded; T5-10 produces the flag-ON gate evidence; T5-11 keeps the interim gate honest until then.

**T5-05 — E-1 population audit (evidence, read-only)** `[evidence] [GATE E-1] [WS-B]`
- **Why/Authority:** E-1 (Stage 2 §30.1), BR-5-046, FDS §13.2/§13.3, §31. Store scope "confirmed only while a writer/population source is identified."
- **Scope/Files:** read-only DB queries + repo writer search; output = evidence record (row counts per hotel, recency (max created/updated), writer identification) for `room_inventory`, `out_of_order`, `out_of_service`, `rate_restrictions`, `close_to_arrival`, `close_to_departure`, `zero_sell_limits`, `room_type_sell_limits`, `minimum_length_of_stay`, `maximum_length_of_stay`, plus conflict detection for E-2.
- **Preconditions/Dependencies:** none (start immediately; independent of code).
- **Action:** run parameterized read-only queries per store (never mutate); grep repo for writers (`availability-sales.controller.ts:676-707,825-885` known A3 writer; `rate_restrictions` writer = search); record results in the exit evidence (Stage 5 input).
- **Tests/Verification:** evidence completeness checklist (all 10 stores + 2 excluded ones documented); queries themselves reviewed for `hotel_id` scoping.
- **Acceptance:** every store classified: `writer identified + populated` / `writer identified + empty` / `writer UNKNOWN` / `population UNKNOWN`; any store conflicts found triggers E-2 STOP.
- **Risk/Rollback:** read-only; zero risk. Failure = incomplete record ⇒ BLK-P5-01 stays open (never assumed).
- **Completion:** evidence record accepted; feeds T5-07 scope and T5-09 gate.

**T5-06 — E-3 runtime confirmation of the fail-closed chain** `[evidence] [GATE E-3] [WS-B]`
- **Why/Authority:** E-3/U-1; static chain verified in Stage 2 §6.2; runtime observation required before enablement.
- **Scope/Files:** safe environment execution against the **current** stub binding: create attempt with unresolved inputs; observe `AVAILABILITY_ASSERTION_REJECTED` (409) `rejection.code=UNRESOLVED_CAPACITY`, persisted `REJECTED` row, rolled-back transaction, snapshot `0/UNKNOWN`.
- **Preconditions/Dependencies:** test environment with DB (harness DB env); no code change.
- **Action:** execute the observation; record request/response/persisted-state evidence; also record that matrix flag is OFF.
- **Tests/Verification:** evidence shows all four chain links (stub → snapshot → assertion → HTTP) behaving as F-01 documented.
- **Acceptance:** recorded evidence matches static prediction; any deviation ⇒ STOP (contradiction — FDS §3.4).
- **Risk/Rollback:** read-path observation + failed create only; no mutations retained (rollback proven by absence of rows).
- **Completion:** evidence record accepted; closes E-3 for the T5-09 gate.

**T5-07 — Real RestrictionEvaluator implementation** `[create] [BLOCKED: E-1 scope] [WS-B]`
- **Why/Authority:** DS-01 (Option A); BR-5-001…004, BR-5-046; FDS §13.2–13.5. Requirement text is fully specified — this task implements it, nothing more.
- **Scope/Files:** new adapter `apps/api/src/modules/availability/infrastructure/adapters/prisma-restriction.adapter.ts` (implements the existing `RESTRICTION_EVALUATOR` token contract — same interface as `UnresolvedRestrictionAdapter`); store-read helpers (parameterized, `hotel_id` in predicate); provenance structs per FDS §13.4.
- **Preconditions/Dependencies:** **E-1 recorded (T5-05)** fixes the store set; **E-4 pending = store stays excluded** (never guessed).
- **Action:** for inputs `(hotelId, roomType, stayDate, arrivalDate, departureDate[, rateCode/channelCode])`: read each in-scope store hotel-scoped; dimension set `stopSell, cta, ctd, minLos, maxLos, sellLimit, allotmentStopSale (+channelRestriction when a writer exists)`; semantics: no row ⇒ dimension `RESOLVED`(no restriction) + absence recorded in `sources[]` (BR-5-002); read failure ⇒ dimension `UNRESOLVED` + reason (BR-5-001); 2+ stores disagree on a dimension ⇒ `UNRESOLVED` + `sourceConflicts[]` with store+value pairs, **no ranking** (BR-5-003); unproven store ⇒ flagged provenance or excluded (BR-5-046); outcome `RESOLVED` iff every applicable dimension `RESOLVED|NOT_APPLICABLE` (BR-5-004). Preserve existing `UnresolvedRestrictionAdapter` file untouched as rollback artifact.
- **Tests/Verification:** T5-08 suite; typecheck; `hotel_id` predicate review (T5-65 scan).
- **Acceptance:** AC-07/08/09/10/40; INV-P5-09/10/11.
- **Risk/Rollback:** new file only; **not bound** until T5-09 → zero runtime impact. Rollback = never binding it.
- **Completion:** suite green; code reviewed against BR-5-001…004 line-by-line (checklist in exit evidence).

**T5-08 — Evaluator semantics suite (conflict + absence cases required by BLK-P5-01)** `[test] [WS-B]`
- **Why/Authority:** BLK-P5-01 requirement text: "proven by tests incl. conflict and absence cases"; AC-07…AC-10, AC-40.
- **Scope/Files:** `__tests__` for T5-07 adapter (fake/in-memory stores + failure injection).
- **Preconditions/Dependencies:** T5-07.
- **Action:** cover: absence⇒RESOLVED+provenance; unreadable store⇒UNRESOLVED+reason; conflict⇒UNRESOLVED+sourceConflicts (both orders); unproven store flagged/excluded; partial applicability (`NOT_APPLICABLE`); outcome integrity; per-store `hotel_id` scoping assertion; `channel_restrictions` excluded until writer exists; `restrictions/CUTOFF` and `room_inventory` **absent from scope** (T5-47 overlap).
- **Tests/Verification:** this suite; runs in default `pnpm test` (no DB needed — in-memory fakes).
- **Acceptance:** AC-07/08/09/10/40 green; gates T5-09.
- **Risk/Rollback:** test-only.
- **Completion:** suite green, cited in BLK-P5-01 closure evidence.

**T5-09 — DI binding swap to real evaluator (rollback-preserving)** `[modify] [GATE: E-1 + E-3 + T5-07/08] [WS-B]`
- **Why/Authority:** FDS §13.6/§13.7; BR-5-037 condition 1; BLK-P5-01 closure action.
- **Scope/Files:** `apps/api/src/modules/availability/availability.module.ts:26`.
- **Preconditions/Dependencies:** T5-05, T5-06, T5-07, T5-08 all green (BLK-P5-01 evidence complete).
- **Action:** bind `RESTRICTION_EVALUATOR` token to `PrismaRestrictionAdapter`; keep `UnresolvedRestrictionAdapter` class present (rollback artifact, FDS §13.7) with a unit test asserting the stub still compiles/behave-as-designed (its spec `restriction.adapter.spec.ts` stays green in isolation).
- **Tests/Verification:** full API suite with DB env (T5-70/71 battery); harness assertions updated only where they hardcoded `ELIGIBLE` fixtures **as fixtures** (not rules); E-3 chain re-run now expecting **resolved-or-flagged** behavior instead of blanket UNRESOLVED.
- **Acceptance:** chain produces `RESOLVED` outcomes for evidenced empty stores and `UNRESOLVED` for failed/unproven inputs; fail-closed preserved on failure paths.
- **Risk/Rollback:** flag `gba.a3.authoritative` remains OFF (matrix users unaffected); consumers still zero (E1) ⇒ blast radius = server tests. Rollback: revert one binding line → stub restored (previous behavior).
- **Completion:** binding swapped + full battery green + evidence recorded for T5-10.

**T5-10 — Parity soak harness (flag-ON value parity)** `[test] [GATE: T5-09] [WS-B]`
- **Why/Authority:** BR-5-037 condition 4 ("parity soak"); M-4 (matrix migrates by value-sourcing).
- **Scope/Files:** test comparing `GET /availability/matrix` with `gba.a3.authoritative=ON` vs `/availability/snapshot` facts for identical (hotel, roomType, dates), modulo the sanctioned bed-type partition (FDS §12.4 conservation).
- **Preconditions/Dependencies:** T5-09.
- **Action:** run parity across representative fixtures (empty stores, populated restriction rows, blocked, unresolved, bed-type splits); record results as soak evidence.
- **Tests/Verification:** parity test green; conservation (Σ bed cells = room-type figure) asserted (AC-18).
- **Acceptance:** zero unexplained deltas; every delta classified as the disclosed residue rule.
- **Risk/Rollback:** test-only; flag exercised only in test context, never persisted ON (G-5).
- **Completion:** soak evidence recorded — feeds T5-56.

**T5-11 — Interim gate honesty test (until closure)** `[verify] [WS-B]`
- **Why/Authority:** BR-5-009, AC-47, FDS §13.7 (flag OFF + no write-gating cutover + stub rollback artifact retained).
- **Scope/Files:** config-reading test + binding introspection test.
- **Preconditions/Dependencies:** none; runs from Stage 6 start until T5-09 lands (then converts to binding-identity test).
- **Action:** assert `FEATURE_GBA_A3_AUTHORITATIVE` defaults OFF; assert the rollback stub exists; assert no write-gating consumer points at authority while BLK-P5-01 open.
- **Tests/Verification:** config test in default run.
- **Acceptance:** AC-47 green throughout pre-gate period.
- **Risk/Rollback:** test-only.
- **Completion:** test retained permanently as flag-default pin (becomes part of T5-55).

---

## 14. Snapshot / Matrix Consumers (WS-D read surfaces)

Read-surface contracts are verified and made conformant **without changing shapes** (P-17/P-18). Value-sourcing flips only under flag gates.

**T5-19 is the API-side snapshot/reconciliation suite; T5-20 the matrix projection conformance.** (Numbering note: WS-D tasks are T5-19…28 — defined in §16; §14 states the consumer-side read rules they enforce and the frontend hand-off consumed by §20.)

- **Read rule 1 (M-1/AC-16):** after cutover, every display surface reads `/snapshot` (canonical) or `/matrix` (projection) — no third contract (T5-28 enforces).
- **Read rule 2 (P-17/AC-17):** matrix shape never changes; while `gba.a3.authoritative` OFF, `label: 'derived view, not sellable availability'` present and rendered; ON ⇒ values equal authority facts (T5-20 test; T5-40 overlay removal in ON window).
- **Read rule 3 (AC-18):** bed-type partition conserves the authority figure; residue deterministic and disclosed (T5-10 + T5-20).
- **Read rule 4 (AC-19):** `UNKNOWN`/`BLOCKED`/non-empty `unresolvedSources` render as distinct states — enforced server-side by additive fields (T5-02 types) and client-side by T5-30/T5-38 tests.
- **Read rule 5 (BR-5-015/AC-23):** `GET /rates/availability` produces no availability/occupancy figure after disposition (T5-22; frontend stop-reading T5-32).

---

## 15. Backend Consumers (WS-C)

Every consumer found by Stage 1 §6 gets implementation treatment. "Verify" tasks pin compliant behavior; "modify" tasks change MIX/LEG consumers under their gates.

**T5-12 — Reservations write-path coverage pinning** `[verify] [WS-C]`
- **Why/Authority:** FDS §16.1; audit §6.1 table (per-operation verdicts); INV-P5-03/22/24; AC-34 (delete half), AC-41-adjacent.
- **Scope/Files:** `apps/api/src/modules/reservations/repositories/reservation.repository.ts` (`:469/:552`, `:621-625`, `:646-661`, `:732/:748-758`, `:826-869`, `:926-990`, `:889-924`, `:1165-1259`), `change-rate.handler.ts`, `reservations.service.ts:89-93`.
- **Preconditions/Dependencies:** none.
- **Action:** tests asserting: create/status/cancel/no-show/reinstate/batch/waitlist all assert-or-release via the Phase 2 port in-transaction; delete performs **no** availability operation; `CHECKED_OUT` holds (does not release); at most one live assertion per reservation/property; pickup-created reservations gap (DEF-3 dependency, FDS §32) is **documented in test comments as inherited**, not "fixed" here.
- **Tests/Verification:** reservation integration specs (in-memory + harness).
- **Acceptance:** INV-P5-03/22/24 pinned; audit §6.1 AUTH rows stay AUTH.
- **Risk/Rollback:** test-only.
- **Completion:** suite green.

**T5-13 — Front Office upgrade gate re-point** `[modify] [GATE: BLK-P5-01 + DS-03 order] [WS-C]`
- **Why/Authority:** BR-5-019 (authority eligibility; absent-row `true` default never gates a write); FDS §16.2, §20 row "FO upgrade gate"; M-2 (write-gating consumer ⇒ authority operational first); F-05.
- **Scope/Files:** `apps/api/src/modules/front-office/.../upgrade-room.handler.ts:69` (legacy `inventoryDomain.isAvailable` gate) alongside `:85` authority `replaceRoomTypeAssertion`.
- **Preconditions/Dependencies:** BLK-P5-01 closed (T5-05/06/07/08/09); read side sane (T5-30 not strictly required, but §24.2 requires authority operational).
- **Action:** replace the legacy gate with an authority eligibility consult (snapshot/evaluator path scoped to `(hotelId, roomType, stayDate)`); delete the absent-row-`true` default; keep the assertion call as the committing gate; rejection surfaces as typed availability error (§27).
- **Tests/Verification:** unit tests: unavailable ⇒ upgrade rejected deterministically (no silent pass); available ⇒ proceeds; legacy gate no longer referenced (static scan T5-77 family).
- **Acceptance:** AC-24 (FO half); INV-P5-21.
- **Risk/Rollback:** behavior tightening only (stricter gate); rollback = revert commit restoring `isAvailable` call (allowed pre-Phase 11 because legacy service remains).
- **Completion:** tests green + static scan shows no `isAvailable` gate in upgrade path.

**T5-14 — Rates/CRS quote signal re-point + book path retirement readiness** `[modify] [GATE: BLK-P5-01] [WS-C]`
- **Why/Authority:** BR-5-016/017/050; FDS §20 rows `quote` RE-POINT / `book` RETIRE; F-07; M-2.
- **Scope/Files:** `apps/api/src/modules/rates/crs-engine.service.ts` (quote availability/blockReason signal `:149-163`), `GET /rates/engine/quote` handler; frontend caller `apps/web/features/reservations/hooks/use-crs-book.ts:87,129` (signal consumption — actual UI edit coordinated with T5-34).
- **Preconditions/Dependencies:** BLK-P5-01 closed.
- **Action:** quote's availability/blockReason signal computed from authority eligibility (pricing/hash retained unchanged — FDS §21.4 explicitly retains pricing); book path: prepare in-repo callers for reservation-create-based booking (the route retirement itself = T5-24).
- **Tests/Verification:** quote tests: blocked/unknown surfaces as signal codes; pricing fields byte-identical for same inputs (no pricing regression test changes).
- **Acceptance:** AC-24 (quote half); INV-P5-21 (`quote.available` no longer the gate).
- **Risk/Rollback:** signal semantics only; rollback = revert (legacy `rate_restrictions` read still present pre-retirement).
- **Completion:** tests green; frontend receives codes not booleans (verified in T5-34).

**T5-15 — GBA consumers verification (pickup consult, cascade, assertion gap)** `[verify] [WS-C]`
- **Why/Authority:** FDS §16.3; audit §6.3; P-21 (`gba.consumers.cascade` ON at deploy); DEF-3 dependency (pickup-created reservations bypass assertion — inherited, documented).
- **Scope/Files:** `pickup-availability.snapshot-consult.ts` (`:25,:66,:148-151`), `prisma-reservation-association.adapter.ts:100-120`, `events.consumer.ts:140-202`, group/allotment aggregates.
- **Preconditions/Dependencies:** none.
- **Action:** tests pinning: pickup consult flag default OFF with fail-closed read when ON; pickup aggregate guards (intra-ledger capacity + stop-sale ordering) unchanged; cascade consumer handles reservation invalidation; **document** pickup-created assertion gap as Phase 4 DEF-3 carry-over (no behavior change — inventing a fix would be a business decision).
- **Tests/Verification:** GBA suites + flag default test.
- **Acceptance:** no GBA behavior change in Phase 5 (scope guard); cascade flag documented ON-at-deploy (T5-54).
- **Risk/Rollback:** test-only.
- **Completion:** suite green.

**T5-16 — Outbound publication guard** `[verify] [WS-C]`
- **Why/Authority:** BR-5-030; DS-07 deferred; INV-P5-30; AC-41.
- **Scope/Files:** `channels/channels.service.ts:158-204` (`pushAvailabilityToChannels(data.available)`), `channel_availability_log`, `POST /channels/availability/push`.
- **Preconditions/Dependencies:** none.
- **Action:** tests + static assertions: no endpoint publishes an availability number sourced from a non-authority computation; `channel_availability_log` feeds no UI/eligibility decision (grep: zero reads outside the channel writer); push retains caller-supplied semantics but may not claim authority (label/test).
- **Tests/Verification:** unit + T5-84-family scan.
- **Acceptance:** AC-41; INV-P5-30.
- **Risk/Rollback:** test-only (push contract itself deferred — no behavior invented).
- **Completion:** guard tests green.

**T5-17 — Reporting/KPI authority sourcing (display half of DS-06)** `[modify] [WS-C]`
- **Why/Authority:** BR-5-028/029; FDS §16.6, §17.4; M-5; AC-42. Availability-labelled figures ⇒ A1; occupancy ⇒ declared reporting (endpoint retained, never labelled availability).
- **Scope/Files:** backend readers feeding dashboards: `reporting-analytics.service.ts:162-209` (occupancy — retained as reporting, source-declared), FO dashboard handlers, command-center widgets' API payloads; authority-sourced availability figures added to whatever payload serves availability-labelled UI (via T5-35 consumption).
- **Preconditions/Dependencies:** BLK-P5-01 for authority-sourced values; labeling/metadata work can prep ungated.
- **Action:** ensure any payload field that is availability-labelled derives from snapshot facts; occupancy fields keep their declared source and hotel scope; no new occupancy definition (DS-06 analytics contract remains deferred — §33).
- **Tests/Verification:** payload contract tests + T5-77 scan (no availability-labelled field sourced elsewhere).
- **Acceptance:** AC-42.
- **Risk/Rollback:** additive fields only; rollback = revert.
- **Completion:** tests green.

**T5-18 — Operational consumers contract verification** `[verify] [WS-C]`
- **Why/Authority:** FDS §16.5 (reconciliation readers, audit-log viewers, restriction editors); BR-5-014/040/041.
- **Scope/Files:** reconciliation UI-less reader path (backend only today), audit-log routes, restriction-edit affordances (frontend portion = T5-33/T5-48).
- **Preconditions/Dependencies:** none (contract-level).
- **Action:** assert readers consume only sanctioned outputs (recon = evidence, logs = audit); editors are callers only (no ownership) — enforced by T5-46/48.
- **Tests/Verification:** route contract tests.
- **Acceptance:** INV-P5-28/29 behaviorally pinned.
- **Risk/Rollback:** test-only.
- **Completion:** green.

---

## 16. API Contracts (WS-D)

**T5-19 — Snapshot + reconciliation contract & isolation suite** `[test] [WS-D]`
- **Why/Authority:** FDS §21.1, §10; BR-5-010/014; AC-13/14; P-1.
- **Scope/Files:** `apps/api/src/modules/availability/api/controllers/availability.controller.ts` (`:12/:20`), `availability-snapshot.service.ts:17-33` (authorize pattern).
- **Preconditions/Dependencies:** T5-02 (types).
- **Action:** e2e tests: exact endpoint set (only snapshot+reconciliation at the authority prefix); 403 without actor; `hotelId='default'` rejected; permission codes enforced; route `:propertyId` governs data access even when header differs (AC-13); every response field of audit §7.1 present; reconciliation output = flagged deltas (no writes).
- **Tests/Verification:** API suite (DB env).
- **Acceptance:** AC-13/14/35 server-side; INV-P5-15.
- **Risk/Rollback:** test-only.
- **Completion:** suite green.

**T5-20 — Matrix projection conformance (shape, label, sourcing)** `[test] [WS-D]`
- **Why/Authority:** FDS §12; P-17/P-18; BR-5-010/011; AC-17.
- **Scope/Files:** `availability-sales.controller.ts` (`:265` route, `:275` flag, `:461-546` sourcing, `:612` label).
- **Preconditions/Dependencies:** none for shape/label tests; flag-ON value test pairs with T5-10.
- **Action:** tests: shape unchanged vs current fixture (snapshot-diff the response); OFF ⇒ label present; ON ⇒ values = snapshot facts; additive eligibility fields optional-but-typed; restriction display fields still raw **until DS-04 read cutover** (test documents the interim state, not a permanent rule).
- **Tests/Verification:** API suite.
- **Acceptance:** AC-17; INV-P5-16.
- **Risk/Rollback:** test-only.
- **Completion:** green.

**T5-21 — DTO validation on all availability writes** `[modify] [WS-D]`
- **Why/Authority:** FDS §21.6 (no `body: any` for writes); BR-5-021(2)/§27 validation row; AC-28.
- **Scope/Files:** `availability-sales.controller.ts` write handlers (`bulk-update :651`, `interval-update :773`, logs `POST :752`, any other availability write) — typed `class-validator` DTOs (existing framework pattern in repo).
- **Preconditions/Dependencies:** none (independent hardening; does not change payloads).
- **Action:** replace `@Body() body: any` with typed DTOs validating exactly the current accepted payload shapes (payload semantics unchanged — the frontend conformance tasks T5-33 align the caller to backend, per FDS §21.5, not vice versa except interval payload which FDS fixes to `ratePlans`+`dateRange`).
- **Tests/Verification:** validation tests: malformed payload ⇒ deterministic validation error **before any DB effect**; valid payloads pass byte-compatibly.
- **Acceptance:** AC-28; INV-P5-18-adjacent (no domain effect on invalid input).
- **Risk/Rollback:** tightening only; rollback = revert DTO.
- **Completion:** tests green; `body: any` gone from availability write routes (scan T5-83).

**T5-22 — `GET /rates/availability` disposition** `[modify] [GATE: E-5 for timing] [WS-D]`
- **Why/Authority:** BR-5-015/051; FDS §21.4 row 1, §20 row 1; G-4 (consumer impact never vetoes); AC-23.
- **Scope/Files:** `apps/api/src/modules/rates-inventory/rates-inventory.service.ts:34-40`, `rates-inventory.controller.ts`.
- **Preconditions/Dependencies:** E-5 (T5-88) for **external timing**; in-repo consumer (T5-32) re-pointed/retired first (M-3 read rule: reads may retire before booking gates but respect E-5).
- **Action:** RETIRE route, **or** de-availability the payload (strip `available`/`occupancyPct`; any retained count gets `hotel_id` filter — removes unscoped read + div-by-zero). Disposition choice = FDS-sanctioned alternatives; plan default = full retirement (404 acceptable per §21.4 precedent T-39), with de-availability as fallback if E-5 reveals an external consumer needing the route for non-availability data.
- **Tests/Verification:** route test (404 or availability-free payload with scoped count); frontend stop-call verified (T5-32); T5-80 scan clean.
- **Acceptance:** AC-23; INV-P5-13/15.
- **Risk/Rollback:** rollback = revert (route returns again); Phase 11 not required (this is read retirement, not table deletion).
- **Completion:** test green + scan clean.

**T5-23 — `/rates/engine/{availability,restrictions}` retirement + quote signal done** `[modify] [GATE: BLK-P5-01 + T5-14] [WS-D]`
- **Why/Authority:** BR-5-016; FDS §21.4 rows 2–3; G-4.
- **Scope/Files:** `crs-engine.service.ts` route handlers; `inventory.domain-service.ts:96-117` (engine availability computation).
- **Preconditions/Dependencies:** T5-14 (signal re-pointed); in-repo consumers switched (T5-32/T5-34 as applicable); BLK-P5-01.
- **Action:** retire both routes as availability sources (404 or stripped); ensure no caller remains (scan).
- **Tests/Verification:** route tests + T5-80 scan.
- **Acceptance:** INV-P5-21; AC-23-family.
- **Risk/Rollback:** revert restores route; readers migrated first ⇒ no user-facing break.
- **Completion:** green + scan clean.

**T5-24 — `POST /rates/engine/book` retirement (booking path)** `[modify] [GATE: BLK-P5-03 (DS-01 + reads disposed)] [WS-D]`
- **Why/Authority:** BR-5-017; FDS §21.4 row 4; audit finding F-07 (raw `INSERT 'CONFIRMED'` bypasses assertion); M-2.
- **Scope/Files:** `crs-engine.service.ts` book handler; `use-crs-book.ts` in-repo caller (frontend switches to reservation create flow via T5-34 coordination).
- **Preconditions/Dependencies:** authority operational (CHAIN-1); quick-book/re-point in place (T5-14, T5-31); E-5 timing.
- **Action:** retire route; booking executes through reservation create command with assertion (already AUTH path); confirm zero in-repo callers (scan) before removal.
- **Tests/Verification:** route 404 test; booking-through-create integration test; T5-80 scan.
- **Acceptance:** INV-P5-03 (no booking bypasses assertion); AC-25.
- **Risk/Rollback:** rollback = revert route (allowed while legacy retained; re-cutover only forward).
- **Completion:** green + scan clean + exit battery.

**T5-25 — `POST /rates/engine/release` retirement (after writers gone)** `[modify] [GATE: §24.4 writer condition] [WS-D]`
- **Why/Authority:** BR-5-018; FDS §21.4 row 6, §20 ("retire once no booking path writes legacy counters"); P-16 (table stays read-only).
- **Preconditions/Dependencies:** T5-24 retired (no book writes counters); T5-50 landed (modify leg removed); last-writer analysis recorded.
- **Action:** retire release route; legacy `availability` table writer set = ∅; table remains read-only until Phase 11 (never dropped — G-1/Phase 11).
- **Tests/Verification:** route test; writer-scan (T5-81) shows zero writes to legacy counters from any path; reconciliation (T5-53/T5-57) shows no drift sources.
- **Acceptance:** INV-P5-21 + P-16 preserved.
- **Risk/Rollback:** revert restores route only if a writer still exists — otherwise dead route; safe.
- **Completion:** green + writer scan clean.

**T5-26 — Error contract typing + code-based detection (server side)** `[verify] [WS-D]`
- **Why/Authority:** BR-5-020; FDS §21.6, §27; AC-27.
- **Scope/Files:** assertion error mapper, availability exceptions, `reservation-availability-wiring.ts`.
- **Preconditions/Dependencies:** T5-04.
- **Action:** assert deterministic codes: `AVAILABILITY_ASSERTION_REJECTED` + `rejection.code ∈ {UNRESOLVED_CAPACITY, CAPACITY_BLOCKED, INSUFFICIENT_CAPACITY, …}`, `IDEMPOTENCY_CONFLICT`, `CURRENT_ASSERTION_MISSING`, `ASSERTION_LINK_MISMATCH`, `CONFLICT`/`OPERATION_IN_PROGRESS`, typed delete-reject error (T5-59); HTTP shapes documented in test fixtures.
- **Tests/Verification:** error-contract tests.
- **Acceptance:** AC-27 server half; INV-P5-32.
- **Risk/Rollback:** test-only (unless a code is currently missing — then add code, behavior-preserving mapping).
- **Completion:** green.

**T5-27 — `/tax-rates` single owner** `[modify] [GATE E-7] [WS-D]`
- **Why/Authority:** BR-5-044; FDS §21.7; AC-29 (route-ownership half); audit finding F-20/F-25.
- **Scope/Files:** `banquet-refs.controller.ts:45`, `availability-sales.controller.ts:131`, `activities.module.ts:24-25`.
- **Preconditions/Dependencies:** E-7 runtime winner recorded (T5-90).
- **Action:** keep the runtime-serving controller as sole declarer; remove the other declaration; no alias route; module registration order documented.
- **Tests/Verification:** single-declaration static test + route smoke test; T5-83 scan.
- **Acceptance:** AC-29; INV-P5-32.
- **Risk/Rollback:** rollback = restore declaration (duplicate returns — degraded but functioning as today).
- **Completion:** green + E-7 evidence cited.

**T5-28 — Fixed-surface gate (no third contract, no invented endpoints)** `[verify] [WS-D]`
- **Why/Authority:** BR-5-010; FDS §21.1/§10 (REQ-10.4); G-8; AC-16.
- **Scope/Files:** API route registry test + static scan.
- **Preconditions/Dependencies:** none; runs at every exit.
- **Action:** assert endpoint set: authority = snapshot+reconciliation (+sanctioned `/availability/*` operational routes per §21.3–21.5, matrix projection, reference data); zero new availability endpoints added during Phase 5; gateway unchanged (no availability-specific routing invented — audit §7.5).
- **Tests/Verification:** route-registry diff test + T5-84-family scan.
- **Acceptance:** AC-16; INV-P5-15.
- **Risk/Rollback:** test-only.
- **Completion:** green at every exit.

---

## 17. Restriction Write Ownership (WS-G, DS-04)

Current writers: A3 `availability-sales.controller.ts:676-707` (bulk) and `:825-885` (interval) via raw `$executeRawUnsafe`; `restrictions (CUTOFF)` writer `:878-885` (excluded); `rate_restrictions` writer unknown (E-1). Target: one authority-owned, parameterized, audited write path; A3 becomes a caller.

**T5-45 — Authority-owned restriction write path** `[create] [WS-G]`
- **Why/Authority:** BR-5-021 (5 duties); FDS §23 REQ-23.1/23.2; INV-P5-12.
- **Scope/Files:** new `apps/api/src/modules/availability/application/services/restriction-write.service.ts` (+ DTOs, + repository with parameterized queries scoped by `hotel_id`); module registration in `availability.module.ts`; audit/provenance record source (who/when/what) writing to the existing audit layer (no new tables — G-1; reuse `/availability/logs` storage semantics or movement-style records per existing schema).
- **Preconditions/Dependencies:** none (buildable ungated — it is not wired to consumers until T5-46).
- **Action:** implement write operations for the six A3-written restriction tables (and `rate_restrictions` when writer identity evidenced): validate payload (typed DTO), enforce `hotel_id` in predicate, write via parameterized queries, invalidate affected availability facts (TR-10.3 analogue — at minimum the A3 matrix read path and any cached availability keys), record provenance/audit row. **Exclude** `restrictions (CUTOFF)` (E-4) and `channel_restrictions` (no writer) explicitly — assert exclusion in tests (T5-47).
- **Tests/Verification:** unit tests for each duty (scope, validation rejection, invalidation called, audit row written, no string-interpolated SQL); T5-65 scoping scan.
- **Acceptance:** AC-30 (path half); INV-P5-12 foundation.
- **Risk/Rollback:** new code, zero callers until T5-46 ⇒ no runtime impact; rollback = don't wire.
- **Completion:** suite green; five duties each have a passing test.

**T5-46 — A3 handlers become callers** `[modify] [GATE: T5-45] [WS-G]`
- **Why/Authority:** BR-5-021 (raw writes prohibited post-cutover); FDS §21.3 (payloads remain operational inputs); REQ-23.3; F-26 (write-side interpolation surface).
- **Scope/Files:** `availability-sales.controller.ts:651` (bulk-update), `:773` (interval-update) handlers.
- **Preconditions/Dependencies:** T5-45; T5-21 (DTO validation already in place).
- **Action:** replace raw `$executeRawUnsafe` bodies with calls to `RestrictionWriteService` preserving the exact request payloads (operational input unchanged — FDS §21.3). CUTOFF-specific write (if any remains in these handlers) is dropped from the authority path per BR-5-024 — **but only the authority path**; if an A3 legacy behavior must persist until E-4, it is explicitly fenced, documented, and statically scanned (not silently removed). Decision point resolved by FDS §23.5: post-cutover raw restriction writes are prohibited ⇒ the handlers stop issuing raw SQL entirely; CUTOFF writes are excluded from the new path (its writer behavior after cutover is: not migrated — recorded as E-4 residual, no behavior invented).
- **Tests/Verification:** handler tests (same request → same domain effect); static scan: zero `$executeRawUnsafe` against restriction tables outside the new service; invalidation triggers on write.
- **Acceptance:** AC-30 full; INV-P5-12.
- **Risk/Rollback:** behavior-preserving refactor; rollback = revert handlers to raw path (dual state allowed pre-cutover; no flag needed because UI wiring is separately gated).
- **Completion:** green + scan clean.

**T5-47 — Excluded-store guard (CUTOFF / channel_restrictions)** `[verify] [WS-G]`
- **Why/Authority:** BR-5-024 (E-4), BR-5-046; FDS §13.2 rows, §23.5; AC-31/40.
- **Scope/Files:** evaluator scope tests (T5-08) + write service tests (T5-45) + static scan.
- **Preconditions/Dependencies:** T5-07, T5-45 (guards both directions).
- **Action:** assert `restrictions (rate_code='CUTOFF')` appears in **neither** evaluator input set nor authority write path nor authority read path; `channel_restrictions` absent until a writer exists; `room_inventory` never an input (BR-5-045).
- **Tests/Verification:** scope assertion tests + T5-81 scan.
- **Acceptance:** AC-31/40; INV-P5-11/31.
- **Risk/Rollback:** test-only.
- **Completion:** green (re-run when E-4 closes — then scope may change only via FDS amendment).

**T5-48 — Restriction-edit affordances wiring (BLK-P5-02 closure)** `[modify] [GATE: T5-45/46] [WS-G]`
- **Why/Authority:** BLK-P5-02 requirement text; BR-5-041 (no silent 404); AC-45; FDS §21.5 wiring condition.
- **Scope/Files:** frontend affordances (audit-log view, interval-update controls — details in T5-33); server side ready after T5-46.
- **Preconditions/Dependencies:** T5-45+T5-46 landed; **before that: affordances visibly disabled** (T5-33 interim state).
- **Action:** enable affordances against canonical routes + authority path; verify disabled-state rendering existed before (test pins the disabled state from Stage 6 start — AC-45 first half).
- **Tests/Verification:** UI test: disabled-not-404 pre-gate; enabled post-gate; endpoint tests.
- **Acceptance:** AC-45 full.
- **Risk/Rollback:** rollback = disable affordance (feature flag not required; UI state reverts) while path stays.
- **Completion:** BLK-P5-02 recorded closed with evidence.

---

## 18. Dual-Write / Cutover (WS-H, DS-05)

**T5-49 — Dual-write status-quo pin (no premature cuts)** `[verify] [WS-H]`
- **Why/Authority:** BR-5-026 (status quo preserved until gates); FDS §24.1/24.3; INV-P5-26.
- **Scope/Files:** `reservation.repository.ts:621` (authority leg) + `:625` (legacy `crs.modifyReservation` leg).
- **Preconditions/Dependencies:** none; active from Stage 6 start.
- **Action:** test asserting **both** legs present while BLK-P5-01/03 open (guards against an over-eager refactor); plus review-gate checklist item at every stage exit ("no write-path reduction occurred").
- **Tests/Verification:** structural test (source assertion or behavioral dual-write observation) + exit checklist.
- **Acceptance:** AC-32 first half green continuously.
- **Risk/Rollback:** test-only.
- **Completion:** test flips expectation at T5-50 (assert single leg) — one-way transition recorded.

**T5-50 — Modify write-leg removal (single write truth)** `[modify] [GATE: BLK-P5-03 = DS-01 operational + DS-03 dispositions] [WS-H]`
- **Why/Authority:** BR-5-025; FDS §24.2/24.4; INV-P5-26; audit F-04.
- **Scope/Files:** `reservation.repository.ts:625` (and any `crs.modifyReservation` availability leg callers for modify flows).
- **Preconditions/Dependencies:** T5-09 (authority operational) + T5-22/23/24 dispositions landed (readers disposed: rates tab T5-32, quote T5-14, engine sources retired) + T5-53 checklist confirming order + T5-49 pin updated.
- **Action:** remove the legacy modify availability leg; modify writes availability only through the assertion port; `extend` (routes through `repo.update`) inherits automatically — verify.
- **Tests/Verification:** modify/extend integration tests: assertion leg only; legacy counter drift stops for modify (reconciliation T5-57 shows modifier-induced drift gone); writer scan (T5-81): `crs.modifyReservation` availability writes = 0.
- **Acceptance:** AC-32 second half; INV-P5-26.
- **Risk/Rollback:** **highest-risk boundary (B-4).** Rollback = revert commit restoring `:625` (legacy service retained until Phase 11 ⇒ genuinely reversible); dual-write drift resumes (monitored, F-04 known).
- **Completion:** tests green + scan clean + reconciliation recorded + exit battery.

**T5-51 — Write-identity audit (F-16 gaps located)** `[verify] [WS-H]`
- **Why/Authority:** BR-5-027 (preserved Phase 2 §5.4; D-15 derivation); audit §6.1 identity gaps: `change-rate.handler.ts:30` (random operation keys, `actorId='system'`), `reservations.service.ts:89-93` + `create-reservation.handler.ts:25,30` (`idempotencyKey` undefined; `hotelId || 'default'` fallback).
- **Scope/Files:** those files + repository identity handling.
- **Preconditions/Dependencies:** none (audit first).
- **Action:** catalog every availability-bearing write: explicit key present? derived per D-15? actor real? `'default'` fallback present? → findings list feeding T5-52.
- **Tests/Verification:** audit table recorded; existing idempotency tests located (`idempotency.test.ts`, `client-idempotency.test.ts`).
- **Acceptance:** gaps enumerated (expected: the three above + any found).
- **Risk/Rollback:** read-only.
- **Completion:** findings list complete.

**T5-52 — Write-identity fixes** `[modify] [GATE: T5-51] [WS-H]`
- **Why/Authority:** BR-5-027: explicit `idempotencyKey` where caller has one; never `hotelId || 'default'` (authority rejects `'default'` anyway — BR-5-032); never silently-minted random operation keys; `actorId` never silently `'system'` where a real actor exists.
- **Scope/Files:** `reservations.service.ts`, `create-reservation.handler.ts`, `change-rate.handler.ts`, repository identity plumbing — per T5-51 findings, using **existing** D-15 derivation helpers where present (Phase 2 rules; no new identity scheme invented).
- **Preconditions/Dependencies:** T5-51.
- **Action:** wire explicit/derived keys and real actors; request-level `Idempotency-Key` header pass-through where the HTTP layer already supports it; deterministic replay semantics preserved (§27 row: same key+hash ⇒ recorded result; different hash ⇒ `IDEMPOTENCY_CONFLICT`).
- **Tests/Verification:** idempotency replay/conflict tests; no-`default`-hotel test; actor propagation test.
- **Acceptance:** BR-5-027 fully evidenced; AC-13 (default rejection) at write entries.
- **Risk/Rollback:** behavior-preserving (keys were undefined ⇒ now defined; collisions surface as conflict errors instead of silent duplicate operations — strictly safer). Rollback = revert.
- **Completion:** tests green; T5-51 findings all closed.

**T5-53 — Ordering checklist (cutover step verification)** `[verify] [WS-H]`
- **Why/Authority:** FDS §24.2 sub-orderings, §24.6; BR-5-025/026/033; S3R-G5.
- **Scope/Files:** exit-gate checklist artifact (this plan §29) — applied at every cutover step's exit.
- **Preconditions/Dependencies:** runs at: CHAIN-1 exit, each T5-22…25 retirement, T5-46, T5-50, T5-56.
- **Action:** record: which gate opened, evidence IDs, which tasks ran, confirmation no earlier-order step was skipped, flag states + their evidence, property-scope regression results.
- **Tests/Verification:** checklist completeness (missing item ⇒ exit fails).
- **Acceptance:** every cutover step has a completed checklist in exit evidence.
- **Risk/Rollback:** process-only.
- **Completion:** all checklists recorded for Stage 5 input (§35).

---

## 19. Legacy Retirement (WS-F)

Per-source disposition is normative in FDS §20/§21.4 (already executed as tasks T5-22…25 under WS-D for routes). WS-F covers the remaining legacy **sources and artifacts**.

| Legacy source | Location | Class | Replacement | Task | Retirement condition | Verification |
|---|---|---|---|---|---|---|
| A3 raw availability computation (matrix OFF path) | `availability-sales.controller.ts:548-581` | PROHIBITED from authority use; interim labeled view only | projection values (flag ON) | T5-20 (interim), T5-40 (ON window), T5-56 (gate) | BLK-P5-01 + parity soak | parity test T5-10 |
| A3 `available` overlay | `:540` (`hasZeroSell ? 0 : …`) | REMOVE — second combination site | authority `fact.sellableAvailable` | **T5-40** `[GATE: flag-ON window]` | flag ON with parity evidence | AC-02 test (overlay expression absent; cells = facts) |
| A3 raw restriction display reads | `:384-405` | RE-POINT → authority projection (DS-04 read side) | authority restriction projection | **T5-39** `[GATE: T5-45/46]` | write path live + projection serving restriction display | BR-5-022 test + scan |
| A3 raw restriction writes | `:676-707`, `:825-885` | RE-POINT → authority write path | T5-45/46 | (T5-46) | T5-45 landed | zero raw SQL scan (T5-81) |
| `restrictions` CUTOFF writer | `:878-885` | DEFERRED — excluded pending E-4 | none (no migration) | T5-47 guard, T5-87 evidence | E-4 documented meaning | AC-31 scope test |
| Legacy `availability` table + `inventory.domain-service` | `schema.prisma:433-453`, `inventory.domain-service.ts:96-117,164,181` | RETAIN read-only (Phase 11) | assertion engine (writers gone) | **T5-41** verify; writers removed via T5-25/T5-50 | Phase 11 owns deletion (never in Phase 5) | writer scan + read-only test; G-1 gate |
| `room_inventory` readers (`availableRooms`) | `availability-sales.controller.ts:62,:249-252` (+ `/availability/room-types`) | PROHIBITED as availability display | authority facts | **T5-44** verify + T5-37 frontend | immediate (BR-5-045) | scan: no `availableRooms` rendered as availability |
| Client-side formulas | audit §5.3 (11 rows) | REMOVE or RE-POINT | server values | T5-30/31/35/36/37 | replacements in place | T5-77 scan clean |
| Dead artifacts | `AvailabilitySales.tsx`, `AnimatedRoomRack.tsx`, `operasales/**`, dead DI (`reservation.repository.ts:15,89`), `PrismaInventoryReservationAdapter`, `PrismaRatesAdapter`, `IInventoryCommitmentPort`, `AttritionCalculationService`, dead client fns | REMOVE — hygiene last | n/a (nothing consumes) | **T5-42** `[GATE G-12]` | behavior-preserving replacements for any dependency landed first (dead = proven) | import-graph test + build green |
| Duplicate routes | `/allotments` vs `/reservations/allotments`, `/group-blocks` vs `/reservations/group-blocks`; `/tax-rates`×2 | CONSOLIDATE | canonical route | **T5-43** (nav dupes) + T5-27 (`/tax-rates`) | after consumers switched (G-12) | route scan |
| Doc drift | stale line citations, counts | FIX | current state | **T5-43** (NB-1…NB-6 chores portion) | after behavior lands (docs cite final state) | citation spot-check (F-23) |

**T5-39 — A3 raw restriction display reads removal** `[modify] [GATE: T5-45/46] [WS-F]`
- **Action/Authority:** replace `:384-405` raw restriction reads serving display with authority projection (BR-5-022); filters live in one place. **Tests:** matrix restriction fields match authority projection; scan shows no raw restriction SELECTs in display path. **Acceptance:** BR-5-022; AC-26-family. **Rollback:** revert to raw reads (flag-ON interim path may keep them until this lands — order recorded in T5-53). **Completion:** green + scan.

**T5-40 — A3 overlay removal (flag-ON window)** `[modify] [GATE: T5-56 flag-ON] [WS-F]`
- **Action/Authority:** delete `hasZeroSell ? 0 : fact.sellableAvailable` combination; cells use authority value (BR-5-023, INV-P5-02, AC-02); restriction-row rendering then follows T5-39. **Tests:** AC-02 (no overlay expression; cell = fact for fixtures incl. zero-sell restriction cases resolved by evaluator). **Rollback:** revert restores overlay (flag OFF users unaffected either way — OFF path keeps label + legacy values until Phase 11? NO — matrix OFF path stays as interim view per P-17; overlay removal applies to the ON path only: implementation detail = guard removal inside ON branch). **Completion:** AC-02 green in ON-mode tests.

**T5-41 — Legacy table read-only/no-new-readers verification** `[verify] [WS-F]`
- **Action/Authority:** P-16, BR-5-018: static scan for new readers of `availability` table added during Phase 5 = 0; legacy readers list unchanged-or-shrunk; `inventory.domain-service` write methods unreferenced after T5-25/T5-50. **Tests:** scan + reference test. **Acceptance:** no new legacy readers (BR-5-018). **Completion:** scan report clean.

**T5-42 — Dead artifact removal (hygiene, gated)** `[hygiene] [GATE: G-12] [WS-F]`
- **Action/Authority:** remove the dead list in the table above **after** confirming zero imports/callers and after any behavior-preserving replacement dependency (M-10). Frontend: `AvailabilitySales.tsx`, `AnimatedRoomRack.tsx`, `operasales/**`, `reservationStore.fetchRoomTypes`, `analyticsStore.fetchOccupancy`, unused `reservation.api.ts:362-407` fns (coordinate with T5-33 which rewrites 405/420/431 — rewrite first, delete only still-dead remainder). Backend: dead DI/adapters. **Tests:** build + import-graph + full suites. **Acceptance:** G-12 order documented per removal (what preceded it). **Rollback:** revert commit. **Completion:** suites green, scans clean.

**T5-43 — Duplicate routes + doc-drift chores (hygiene, gated)** `[hygiene] [GATE: G-12] [WS-F]`
- **Action/Authority:** consolidate nav duplicate routes (after consumers switched); NB-1…NB-6 doc chores + F-23 citation fixes in Phase 5 docs only where behavior changed (never pre-landing). **Tests:** route scan; doc citation spot-check. **Completion:** scans green.

**T5-44 — `room_inventory` display prohibition verification** `[verify] [WS-F]`
- **Action/Authority:** BR-5-045/046, INV-P5-31: scan for `room_inventory`-sourced `availableRooms` rendered anywhere as availability (frontend + `/availability/room-types` payload consumers); any hit requires removal/re-label per AC-23/42 rules (payload may exist as legacy metadata but may not feed availability display). **Tests:** scan + payload-consumer test on AvailabilityPage room-type rendering. **Acceptance:** INV-P5-31. **Completion:** scan clean.

---

## 20. Frontend Migration (WS-E)

No UI/UX redesign (brief §2 scope): migrations re-point data sources and surface states; layouts/copy of consequence are out of scope except where the spec mandates surfacing (label, UNKNOWN/BLOCKED states) or removes illegal content (client math, mocks).

**T5-29 — `useAvailabilitySnapshot` hook + shared type adoption** `[create] [WS-E]`
- **Why/Authority:** FDS §17.1/§17.2 (sanctioned sources, hooks); DS-02; audit E1 (zero consumers) → this is the first sanctioned consumer; AC-21.
- **Scope/Files:** `apps/web/features/availability/hooks/use-availability-snapshot.ts` (React Query — the repo standard for new features), consuming T5-02 shared types; property id from route context (AC-13 frontend half).
- **Preconditions/Dependencies:** T5-02. (Build ungated; **switching data on** is gated — hook's consumers activate under T5-30.)
- **Action:** hook with `queryKey` per `(propertyId, roomType, arrival, departure, channel, rate)`, standard `staleTime`, error state exposing `AVAILABILITY_ASSERTION_REJECTED`-style codes for mutation paths (T5-34); no caching claims beyond `LIVE_READ` honesty (BR-5-039).
- **Tests/Verification:** hook unit test (fetch args, property scoping, error mapping); typecheck.
- **Acceptance:** AC-21; M-1 satisfied for future consumers.
- **Risk/Rollback:** additive; revert harmless.
- **Completion:** test green, exported.

**T5-30 — AvailabilityPage cutover (authority-backed display)** `[modify] [GATE: BLK-P5-01] [WS-E]`
- **Why/Authority:** FDS §17.3/§17.7; BR-5-011 (eligibility surfacing); BR-5-012 (remove client math — F-02/F-03 rows); M-1; AC-01/17/19/20.
- **Scope/Files:** `apps/web/features/reservations/workspace/pages/AvailabilityPage.tsx` (`:851-876` occupancy math, `:1006-1015` mislabelled `totalAvail` Σphysical, grid sourcing), `reservation.api.ts:232` matrix call stays for shape, new snapshot consumption where needed for eligibility.
- **Preconditions/Dependencies:** CHAIN-1 closed (BLK-P5-01); T5-29; T5-02.
- **Action:** grid values from matrix projection (flag-ON authoritative post T5-56; interim label rendered while OFF — `label` is currently invisible here); **remove** `occupancy = (physical−available)/physical` and `totalAvail = Σ physicalInventory` client formulas (display occupancy only from declared reporting source per T5-35, or hide); render `bookingEligibility` UNKNOWN/BLOCKED as distinct states; unresolved cells never bare numbers.
- **Tests/Verification:** component tests: UNKNOWN/BLOCKED render distinctly (AC-19), no client-side availability arithmetic present (static), label visible in OFF state (AC-17), values match projection fixture.
- **Acceptance:** AC-01 (no disagreement with snapshot), AC-17/19/20.
- **Risk/Rollback:** rollback = revert to previous page (legacy matrix OFF path still live pre-Phase 11) ⇒ user-visible prior behavior.
- **Completion:** tests green; F-02/F-03 closed.

**T5-31 — Quick-book matrix cell mapping** `[modify] [GATE: BLK-P5-01] [WS-E]`
- **Why/Authority:** F-13 (cells hardwired `available: 0`, `AvailableRatesMatrix.tsx:61-89`); F-10; BR-5-011/012; AC-22.
- **Scope/Files:** `apps/web/features/.../quick-book/AvailableRatesMatrix.tsx` (`:61-89`, render `:270,294,300,306`, label render `:140-142` already exists).
- **Preconditions/Dependencies:** projection contract stable (T5-02/T5-20); BLK-P5-01 for authoritative values.
- **Action:** map cells from projection (value + eligibility + label) instead of literal `available: 0`; booking click consults eligibility state (UNKNOWN ⇒ not bookable; BLOCKED ⇒ shown blocked).
- **Tests/Verification:** mapping test over fixture cells; static: no literal `available: 0` in file.
- **Acceptance:** AC-22; F-13 closed.
- **Risk/Rollback:** revert restores broken-by-design zeros (prior behavior — known broken, not working).
- **Completion:** green.

**T5-32 — Rates-inventory tab re-point/retirement** `[modify] [GATE: T5-22 disposition] [WS-E]`
- **Why/Authority:** F-04 (tab calls `GET /rates/availability`, heading collides with authority); BR-5-015; AC-23; M-3.
- **Scope/Files:** `apps/web/app/(dashboard)/rates-inventory/page.tsx:143,155,312-351`.
- **Preconditions/Dependencies:** T5-22 decision in-flight (coordinate: retire call before backend retires route).
- **Action:** remove/replace the "Availability Snapshot" tab's availability figures with authority data (snapshot via T5-29) **or** remove the tab's availability section if the surface is purely legacy; pricing parts of the page unaffected; stop calling `/rates/availability`.
- **Tests/Verification:** no call to `/rates/availability` from web (T5-80 scan); component test if figures retained (must come from authority).
- **Acceptance:** AC-23 client half; name collision resolved.
- **Risk/Rollback:** revert restores call — only safe while route exists; order recorded in T5-53.
- **Completion:** scan clean.

**T5-33 — Logs/interval-update UI conformance (DS-11 part 1 frontend)** `[modify] [GATE: affordance portion → T5-48] [WS-E]`
- **Why/Authority:** BR-5-040/041; FDS §21.5 (canonical routes + payloads); F-08 (broken `:405/:420/:431`); AC-26/45.
- **Scope/Files:** `reservation.api.ts:405` (`/availability/matrix/logs` → `/availability/logs`), `:420` (POST same), `:431` (`POST /activities/availability/interval-update` → `/availability/interval-update` with payload `ratePlans` + `dateRange{start,end}` replacing `ratePlanCodes/startDate/endDate`), paging params reconciled to backend (`pageSize` vs backend contract), `AvailabilityPage.tsx:309` (`intervalUpdate` caller), `:545` (`getMatrixLogs` caller).
- **Preconditions/Dependencies:** T5-21 (backend DTOs define the contract).
- **Action:** conform callers to canonical routes/payloads (fixes two silent 404s); **restriction-editing affordances** (interval controls, bulk-edit UI if present) rendered visibly disabled until T5-48 (AC-45 first half) — audit-log *viewing* is read-only and may enable immediately.
- **Tests/Verification:** API client tests asserting exact routes/payloads; UI test: edit affordance disabled-not-404 pre-gate; log view loads from canonical route.
- **Acceptance:** AC-26 (all four bullets: logs GET/POST, interval payload, bulk unchanged, paging reconciled); AC-45 interim.
- **Risk/Rollback:** fixes broken calls — strictly better; rollback = revert (back to 404s).
- **Completion:** tests green; zero calls to `/availability/matrix/logs` or `/activities/availability/*` (T5-80).

**T5-34 — Frontend error handling by code** `[modify] [GATE: T5-14/T5-24 coordination] [WS-E]`
- **Why/Authority:** BR-5-020; AC-27; F-27 (substring detection prohibition).
- **Scope/Files:** `use-crs-book.ts` (quote/book error paths), reservation mutation hooks, any `err.message.includes(...)` in availability paths.
- **Preconditions/Dependencies:** T5-04/T5-26 (codes defined).
- **Action:** replace message-substring matching with `error.code` / `rejection.code` matching (`AVAILABILITY_ASSERTION_REJECTED`, `UNRESOLVED_CAPACITY`, `CAPACITY_BLOCKED`, `INSUFFICIENT_CAPACITY`); surface deterministic messages per code.
- **Tests/Verification:** unit tests per code; static scan for `.includes(` on availability errors.
- **Acceptance:** AC-27 full.
- **Risk/Rollback:** behavior-preserving mapping; revert on mismatch.
- **Completion:** green + scan clean.

**T5-35 — KPI/occupancy surfaces sourcing** `[modify] [GATE: BLK-P5-01 for authority figures] [WS-E]`
- **Why/Authority:** BR-5-028/029; M-5; AC-42; audit F-05/F-06/F-08/F-09 (hardcoded `78.4%` mock).
- **Scope/Files:** `apps/web/app/(dashboard)/layout.tsx:108-115`, `front-office/components/dashboard/KpiHeader.tsx:30-37`, `command-center/ChartsRow.tsx:61-101` + `KpiRow.tsx:13-20`, `reporting-analytics/page.tsx:14` (mock), `packages/ui-web/.../RoomGrid.tsx:118-131` (client status counting).
- **Preconditions/Dependencies:** T5-17 payloads; authority values for availability-labelled fields.
- **Action:** availability-labelled figures come from authority-backed payloads (or are labelled non-authoritative until then — never bare numbers claiming live truth); occupancy from declared reporting sources (kept as occupancy, never relabelled availability); remove hardcoded mock `78.4%` presentation of live-looking availability/occupancy (either mark as demo data per existing patterns or source it — decision fixed by AC-42: no mock presented as live truth).
- **Tests/Verification:** payload source tests; static: mock literal no longer rendered as live KPI.
- **Acceptance:** AC-42.
- **Risk/Rollback:** display-source change; revert restores previous sourcing (same prior behavior).
- **Completion:** green.

**T5-36 — GBA/ledger views: server values, no client recomputation** `[modify] [WS-E]`
- **Why/Authority:** M-6; BR-5-012; AC-20; audit F-11/F-12 (`GroupBookingsListView`, `GroupBookingDetailView`, `AllotmentDetailView` client math).
- **Scope/Files:** `apps/web/features/group-allotment/views/GroupBookingsListView.tsx:99-101,154`, `GroupBookingDetailView.tsx:96-114,418,815-819`, `AllotmentDetailView.tsx:687,757,764`.
- **Preconditions/Dependencies:** server ledger fields already exist in GBA payloads (Phase 4 read models) — verify first (sub-task: payload audit; if a value is genuinely absent server-side, **surface existing ledger values, never invent client math**; adding a server field for an existing ledger value is data exposure, not a new rule).
- **Action:** replace `picked/blocked` %, `quota − picked − released`, pickup/wash client formulas with server-provided ledger fields; hardcoded `$420` rate mock treated like T5-35 mock rule.
- **Tests/Verification:** component tests with server fixtures; static scan (T5-77/82) shows no availability/quota arithmetic in these files.
- **Acceptance:** AC-20 (GBA rows); M-6.
- **Risk/Rollback:** revert to client math (prior behavior) if payload gap found — escalate as evidence note, not assumption.
- **Completion:** green + scan clean.

**T5-37 — Shadow-state and fake-default cleanup** `[modify] [GATE: G-12 for removals] [WS-E]`
- **Why/Authority:** audit §5.4 (shadow stores), F-09-adjacent (`hotelStore.totalRooms = h.total_rooms || 200` hardcoded 200); BR-5-045 (no `availableRooms` display); INV-P5-34 (no fake certainty).
- **Scope/Files:** `reservationStore.roomTypeList` (`:114-115,172,187-191` — strip `availableRooms` consumption from `/availability/room-types` display paths), `analyticsStore` occupancy fallback feeding layout, `hotelStore.ts:123,148` fake `200` default, `commandCenterStore` occupancy gauge shadow.
- **Preconditions/Dependencies:** T5-30/35 replacements in place before removing (G-12).
- **Action:** remove fake defaults from availability/occupancy displays (a `200`-room default may never render as live inventory); shadow copies no longer authoritative inputs; navigation-only store keys untouched (OK per audit).
- **Tests/Verification:** tests: no `|| 200` in rendered room counts; shadow fields not read by migrated surfaces.
- **Acceptance:** INV-P5-34; part of AC-20.
- **Risk/Rollback:** revert restores fallback display (prior behavior) — only post-replacement anyway.
- **Completion:** green.

**T5-38 — Frontend availability test coverage (new)** `[test] [WS-E]`
- **Why/Authority:** audit §5.8 (zero availability tests); DS-09 evidence policy; §36.2 web gate.
- **Scope/Files:** new tests under `apps/web` colocated with features: surfacing (UNKNOWN/BLOCKED/label), matrix mapping (T5-31), snapshot hook (T5-29), error codes (T5-34), no-client-math static assertions (as tests).
- **Preconditions/Dependencies:** tasks above as they land.
- **Action:** author suites; wire into `pnpm test` for `apps/web`.
- **Tests/Verification:** `cd apps/web && pnpm test` green (excluding documented baseline suite).
- **Acceptance:** availability-covered test count > 0 (closes §5.8 gap); AC-19/22/27 client evidence.
- **Risk/Rollback:** test-only.
- **Completion:** suites green in exit battery.

---

## 21. Logs / Audit / Reconciliation (WS-J, DS-11)

**T5-57 — Reconciliation read-only + flagged-facts suite** `[test] [WS-J]`
- **Why/Authority:** BR-5-014; FDS §25; INV-P5-28; TR-15.5 (legacy never wins); AC-35.
- **Scope/Files:** `availability.controller.ts:20` + reconciliation service.
- **Preconditions/Dependencies:** none.
- **Action:** tests: endpoint performs zero writes (DB-state snapshot before/after); discrepancies reported as flagged deltas; authority value used wherever downstream logic consumes the result; `automaticRepair` never true in this path (GBA detectors `gba.reconciliation.enabled` remain separate, read-only).
- **Tests/Verification:** API suite (DB env).
- **Acceptance:** AC-35; INV-P5-28.
- **Risk/Rollback:** test-only.
- **Completion:** green.

**T5-58 — Audit integrity verification** `[verify] [WS-J]`
- **Why/Authority:** FDS §26.2/§26.5; Phase 2 §5.1/5.2 (append-only movements, balances = Σ movements); INV-P5-23/29.
- **Scope/Files:** DB trigger on `availability_assertion_movements`, `reservation_availability_operations`, balance tables; `/availability/logs` routes.
- **Preconditions/Dependencies:** none (schema exists — no changes, G-1).
- **Action:** tests: direct balance UPDATE rejected by trigger; movements insert-only; every availability mutation path leaves an audit record with identity/actor/hotel/date; log reads scoped by `hotel_id`.
- **Tests/Verification:** harness tests (DB env).
- **Acceptance:** AC-36; INV-P5-23.
- **Risk/Rollback:** test-only.
- **Completion:** green.

**T5-59 — Delete-reject-when-availability-state** `[modify] [WS-J]`
- **Why/Authority:** BR-5-043 (deterministic typed business error before any mutation); FDS §26.4(2); AC-34; schema intent `schema.prisma:17399,:17426` (`onDelete: Restrict`).
- **Scope/Files:** `reservation.repository.ts:889-924` (terminal-only delete path) + delete command handler.
- **Preconditions/Dependencies:** T5-26 (typed error contract).
- **Action:** pre-mutation existence check for `reservation_availability_state`/`reservation_availability_operations` rows for the reservation ⇒ reject with the typed business error (defined in T5-26); never surface a raw FK error; zero partial mutation (transaction proves no-op). Operator guidance (retain record) documented in the error message payload.
- **Tests/Verification:** tests: delete with state ⇒ typed rejection + zero row changes; delete without state ⇒ success + no availability operation (pairs with BR-5-042 pin T5-60).
- **Acceptance:** AC-34 full; INV-P5-22.
- **Risk/Rollback:** behavior tightening (previously DB-restrict error ⇒ now clean domain error); rollback = revert (Restrict constraint still protects).
- **Completion:** green.

**T5-60 — Delete never releases (pin)** `[test] [WS-J]`
- **Why/Authority:** BR-5-042 (P-8/P-9, Decision 12 preserved); INV-P5-22.
- **Scope/Files:** delete path + assertion engine.
- **Preconditions/Dependencies:** T5-59 (same path).
- **Action:** test: terminal delete performs no release/assert; capacity attributable to completed stay retained; audit rows not cascade-deleted from reservations path (static + runtime).
- **Tests/Verification:** integration test.
- **Acceptance:** AC-34 delete-half; INV-P5-22.
- **Risk/Rollback:** test-only.
- **Completion:** green.

**T5-61 — Logs-are-never-authority guard** `[verify] [WS-J]`
- **Why/Authority:** FDS §26.2 four-layer rule; INV-P5-29; G-8 family.
- **Scope/Files:** static scan + review of read paths.
- **Preconditions/Dependencies:** none.
- **Action:** scan: no availability figure computed from log/audit/reconciliation tables; no write path mutates domain state from an audit record; dashboards don't read `/availability/logs` as data source.
- **Tests/Verification:** scan report + spot tests.
- **Acceptance:** INV-P5-29.
- **Risk/Rollback:** review-only.
- **Completion:** clean report at exit.

---

## 22. Feature Flags (WS-I, DS-10)

**T5-54 — Flag + test-env documentation duties** `[config] [WS-I]`
- **Why/Authority:** BR-5-034/036; FDS §28.2 (Stage 4 configuration duty); audit E3 (`FEATURE_*` absent from env files).
- **Scope/Files:** `compose.yaml`, `.env.example` (docs only), plus test/CI documentation for `AVAILABILITY_TEST_DATABASE_URL`.
- **Preconditions/Dependencies:** none.
- **Action:** declare all seven flags with defaults and semantics: `FEATURE_GBA_A3_AUTHORITATIVE` (OFF), `FEATURE_GBA_CONSUMERS_CASCADE` (ON at deploy), `FEATURE_GBA_WASH_SCHEDULER_ENABLED` (OFF, blocked A+B), `FEATURE_GBA_RECONCILIATION_ENABLED` (OFF/soak), `FEATURE_GBA_PICKUP_TWO_LAYER_CONSULT` (OFF), `FEATURE_GBA_PICKUP_CANONICAL_READ` (OFF→sequence), `FEATURE_GBA_PICKUP_CANONICAL_WRITE` (OFF→sequence); document `AVAILABILITY_TEST_DATABASE_URL` requirement for DB suites. **No code changes; no flag values changed** (G-5).
- **Tests/Verification:** doc presence test (T5-55); diff review shows config-docs only.
- **Acceptance:** AC-37 (documentation half); BR-5-036.
- **Risk/Rollback:** docs only.
- **Completion:** files updated; cited in exit evidence.

**T5-55 — Flag mechanism verification suite** `[test] [WS-I]`
- **Why/Authority:** BR-5-035/037/038; AC-37/39; G-5; P-21.
- **Scope/Files:** `config.service.ts:63-65`, flag call sites (15 sites/7 flags), platform `feature-flag.service.ts` (must hold zero availability flags).
- **Preconditions/Dependencies:** none.
- **Action:** tests: all seven flags read via `ConfigService.getFeatureFlag` with default OFF except documented deploy-invariant (`gba.consumers.cascade` ON documented as deploy config, default-off code reading preserved); zero availability flags on the platform DB system; no `NEXT_PUBLIC_*` availability flags; wash flag existence documented as blocked (Deviation A/B, §32).
- **Tests/Verification:** config unit tests + platform-flag scan.
- **Acceptance:** AC-37/39; BLK-P5-05 evidence.
- **Risk/Rollback:** test-only.
- **Completion:** green.

**T5-56 — `gba.a3.authoritative` ON gate execution (operational)** `[ops] [CUTOVER GATE: BLK-P5-01 evidence] [WS-I]`
- **Why/Authority:** BR-5-037 four conditions; FDS §28.3; AC-38.
- **Scope/Files:** environment configuration (operational action in Stage 6+ runtime — never a code side effect).
- **Preconditions/Dependencies:** T5-09 (evaluator bound), T5-05 (E-1 recorded), T5-06 (E-3 recorded), T5-10 (parity soak green) — all four recorded in exit evidence.
- **Action:** flip flag ON in target environment with recorded evidence per condition; verify matrix serves authority values; T5-40 overlay removal proceeds in the ON window; T5-30/31 values now authoritative.
- **Tests/Verification:** post-ON smoke: matrix = snapshot parity on sample scopes; label removed? (label semantics: ON state = authority-backed; interim label is the OFF-state marker per P-17 — rendered while OFF).
- **Acceptance:** AC-38.
- **Risk/Rollback:** **rollback = flag OFF** (labeled legacy view restored — decision-sanctioned, FDS §24.7).
- **Completion:** gate record signed in exit evidence; feeds Stage 5 (§35).

---

## 23. Database / Schema (WS-K)

**Planned schema impact: NONE.** G-1 (user constraint + Phase 4 Deviation C) prohibits schema, migration, index, constraint, seed, and backfill work in Phase 5. Legacy `availability` table deletion belongs to Phase 11 (P-16). WS-K exists to **prove** this holds.

**T5-62 — Zero-schema-change gate** `[gate] [WS-K]`
- **Why/Authority:** G-1; FDS §2.2/§36.3; Phase 4 Deviation C; INV-P5-34-adjacent scope integrity (AC-44).
- **Scope/Files:** `packages/db/schema.prisma`, `packages/db/migrations/**` — watched, not edited.
- **Preconditions/Dependencies:** none; runs at **every** stage exit.
- **Action:** assertions: `git status` shows zero changes under `packages/db/**` attributable to Phase 5; migration count unchanged (51 finished + 2 historical unfinished per NB-4 — baseline recorded); `npx prisma validate` (main + inventory) green with only the 2 pre-existing SetNull warnings; `prisma migrate diff` empty vs baseline.
- **Tests/Verification:** gate commands in exit battery (§34).
- **Acceptance:** AC-44 (schema half); G-1.
- **Risk/Rollback:** detection-only; a hit fails the exit and is reverted before proceeding.
- **Completion:** green at every exit.

**T5-63 — Schema-evidence citation currency check** `[verify] [WS-K]`
- **Why/Authority:** F-23 (never cite stale lines as live); Stage 3 §36.2 documentation consistency duty.
- **Scope/Files:** citations in this plan and the FDS that implementation will rely on (`schema.prisma:433-453` legacy availability; `:17399/:17426` Restrict intents; assertion tables ownership).
- **Preconditions/Dependencies:** none; run before tasks that read those lines (T5-41, T5-59, T5-07).
- **Action:** confirm cited lines still match local source (no drift since Stage 1/3 evidence); if drifted, record the delta in exit evidence and update **this plan's** citations (documentation fix — allowed; schema untouched).
- **Tests/Verification:** spot-check report.
- **Acceptance:** all cited schema anchors current.
- **Risk/Rollback:** docs-only.
- **Completion:** report filed.

---

## 24. Property Isolation (WS-L)

Isolation is a cross-cutting contract (FDS §22 REQ-22.1…22.8); WS-L makes it executable across reads, writes, jobs, logs, reconciliation, API, and frontend.

**T5-64 — Availability API isolation suite** `[test] [WS-L]`
- **Why/Authority:** BR-5-032/047; FDS §22.1/22.3/22.4; AC-13.
- **Scope/Files:** `availability.controller.ts`, `availability-snapshot.service.ts:17-33`, `property-scope.guard.ts`, `partition-router.middleware.ts`.
- **Preconditions/Dependencies:** none.
- **Action:** tests: missing context ⇒ 403 (authority routes); `hotelId='default'` ⇒ rejected at entry; non-member property ⇒ 403; permission codes enforced; **route `:propertyId` ≠ header/tenant scope ⇒ route wins for both authz and data** (the F-19 divergence closed); reconciliation similarly scoped.
- **Tests/Verification:** API suite (DB env, two-property fixture).
- **Acceptance:** AC-13 full; INV-P5-14.
- **Risk/Rollback:** test-only.
- **Completion:** green.

**T5-65 — Raw SQL hotel-scoping scan** `[verify] [WS-L]`
- **Why/Authority:** BR-5-049 (TR-14.1); FDS §22.6; G-7; INV-P5-13.
- **Scope/Files:** all raw SQL touched by Phase 5 work: `availability-sales.controller.ts` (bulk/interval/logs), new `restriction-write.service.ts`, `prisma-reservation-association.adapter.ts` (availability-adjacent statements), `inventory.domain-service.ts`.
- **Preconditions/Dependencies:** none; re-run after T5-45/46, T5-50.
- **Action:** every statement's predicate includes `hotel_id`; flag any bare-id mutation; new service parameterized + scoped (pair with T5-45 duties).
- **Tests/Verification:** scan report + unit scoping test on the new write service (cross-hotel write attempt fails).
- **Acceptance:** AC-14; INV-P5-13.
- **Risk/Rollback:** detection-only.
- **Completion:** clean report.

**T5-66 — `@PropertyScope(false)` equivalence check** `[verify] [WS-L]`
- **Why/Authority:** BR-5-048 (F-19); FDS §22.5.
- **Scope/Files:** `availability-sales.controller.ts:30-31`, `banquet-refs.controller.ts:9-10`, the snapshot `authorize()` pattern (`availability-snapshot.service.ts:17-33`).
- **Preconditions/Dependencies:** T5-64 (pattern established).
- **Action:** verify each `@PropertyScope(false)` controller touching availability-bearing data performs equivalent scoped authorization (membership + permission codes + `default` rejection + route/header reconciliation) — or gate its availability routes behind the pattern; `requireHotelId()` alone (header-derived) is insufficient where a route property exists.
- **Tests/Verification:** per-controller authz tests; review note per controller.
- **Acceptance:** BR-5-048 evidenced per controller; any gap becomes a scoped fix task attached to this ID (fix = authz code, no business rule).
- **Risk/Rollback:** tightening authz; rollback = revert per controller.
- **Completion:** all availability-bearing controllers pass or are fixed under this task.

**T5-67 — Jobs/workers/invalidation scoping** `[verify] [WS-L]`
- **Why/Authority:** FDS §22.8 (leakage prevention across async paths); audit `events.consumer.ts:140-202` (invalidation deletes `availability:${hotelId}:*`).
- **Scope/Files:** `events.consumer.ts`, any availability-touching BullMQ/Temporal jobs, wash/reconciliation schedulers (read paths only).
- **Preconditions/Dependencies:** none.
- **Action:** verify every async path carries explicit `hotelId` (never ambient/global); invalidation patterns hotel-prefixed; no job aggregates availability across hotels; the known no-op invalidation key (F-14) documented, not "fixed" (fixing cache keys = TR-10.3 territory, out of Phase 5).
- **Tests/Verification:** job unit tests with two hotels; scan.
- **Acceptance:** INV-P5-13 across async paths.
- **Risk/Rollback:** test-only.
- **Completion:** green.

**T5-68 — Frontend property context** `[verify] [WS-L]`
- **Why/Authority:** FDS §22.1 (frontend passes route propertyId); AC-13 client half.
- **Scope/Files:** T5-29 hook, `reservation.api.ts` availability calls, AvailabilityPage route param flow.
- **Preconditions/Dependencies:** T5-29.
- **Action:** tests: authority calls carry the active property id from route/session context; no hardwired property, no cross-property query reuse (queryKey includes propertyId).
- **Tests/Verification:** hook/component tests.
- **Acceptance:** AC-13 full (client+server).
- **Risk/Rollback:** test-only.
- **Completion:** green.

---

## 25. Test Strategy (WS-M)

**Levels** (executed in Stage 6; defined here):

| Level | Scope | Command (per AGENTS.md) | DB requirement |
|---|---|---|---|
| Unit | evaluator semantics, snapshot calculator, state pins, identity, DTOs, hooks | `cd apps/api && pnpm test` (focused patterns) / `cd apps/web && pnpm test` | no |
| DB integration | assertions, delete-reject, audit triggers, scoping, reconciliation | API jest with `AVAILABILITY_TEST_DATABASE_URL` set (48 `*.postgres.spec.ts` inventory) | **yes** |
| API contract | snapshot/reconciliation/matrix/logs/interval/errors/isolation | API suite subset | yes |
| Frontend | surfacing, mapping, hooks, error codes, no-client-math | `cd apps/web && pnpm test` + typecheck + lint | no |
| GBA regression | cascade, pickup consult, aggregates untouched | GBA battery (79 suites baseline) | yes |
| Property isolation | two-property fixtures across all of the above | dedicated suite + WS-L tasks | yes |
| Concurrency/idempotency | replay/conflict, CONFLICT/OPERATION_IN_PROGRESS, dual-write pin | focused specs | yes |
| Static (grep) gates | WS-N scans | scripted scans (see §26) | no |

**Mandatory scenario families** (map to ACs in §27): fail-closed floors; blocked vs unknown vs zero; conflict + absence evaluator cases; isolation (route/header/default); error codes; delete semantics; matrix shape/label/parity; bed-type conservation; idempotency replay/conflict; flag defaults; reconciliation read-only; delete-with-state rejection; quick-book mapping; eligibility surfacing.

**T5-69 — Unit suite assembly** `[test]` — author/run unit groups (T5-01/03/08/55 pins + DTO + identity). Completion: green in battery. **Rollback:** n/a.
**T5-70 — DB integration battery** `[test]` — run all 48 postgres suites with `AVAILABILITY_TEST_DATABASE_URL`; **zero harness-skips permitted** for that reason (BR-5-033; closes F-17 first half). Completion: recorded run, 0 skip-attributable-to-env.
**T5-71 — API contract battery** `[test]` — authority + projection + restriction routes + error contract + isolation suites. Completion: green (baseline failures excluded as documented).
**T5-72 — Frontend battery** `[test]` — `apps/web` typecheck (0), lint (exit 0), test (T5-38 suites green; baseline suite `reservation-actions.test.ts` failures preserved as-is). Completion: recorded.
**T5-73 — GBA regression battery** `[test]` — GBA 79-suite baseline stays green (Phase 5 touches none of its semantics by scope — any delta = investigation, not redefinition). Completion: green.
**T5-74 — Property isolation regression** `[test]` — WS-L suites as one grouped run. Completion: green.
**T5-75 — Concurrency/idempotency** `[test]` — T5-52 replay/conflict tests, dual-write pin (T5-49), assertion conflict semantics (`CONFLICT`/`OPERATION_IN_PROGRESS` rows of §27). Completion: green.
**T5-76 — E-8 baseline reconciliation (first executed exit)** `[evidence] [GATE E-8]` — full API run vs §6 baselines (178/1470/1skip/6fail; known 2-suite/6-test failures; sole `t45` skip; web 1-suite/10-test baseline) with deviations **explained, never redefined**; DB env set; flags reported with gate evidence (§36.2). **Completion:** reconciliation table filed — entry criterion for Stage 5 (§35).

---

## 26. Static Analysis Strategy (WS-N)

Ten scan families as scripted grep/rg gates, run at every code-changing exit (paired with exit battery). Each maps to a prohibition.

| Task | Scan family | Pattern (illustrative) | Prohibition closed | Failure action |
|---|---|---|---|---|
| **T5-77** | legacy availability arithmetic | client: `AvailabilityPage.tsx` occupancy/totalAvail formulas, `KpiHeader`, `RoomGrid`, group/allotment % math, `hotelStore \|\| 200`; backend: `rates-inventory.service.ts:34-40`, `inventory.domain-service.ts:96-117` reachable-from-new-code | BR-5-012/015/028, AC-20/23 | fix or justify-by-retirement note |
| **T5-78** | duplicate authority / second combination | `hasZeroSell ? 0`, `sellableAvailable` computed outside authority, `available:` literals in web, new availability endpoints | BR-5-008/010/023, G-8, AC-02/16 | remove; route registry test fails |
| **T5-79** | unscoped queries | availability/restriction SQL without `hotel_id` in predicate | BR-5-031/049, G-7, AC-14 | add scope or revert |
| **T5-80** | legacy route usage (frontend) | calls to `/rates/availability`, `/rates/engine/*` (retired set), `/availability/matrix/logs`, `/activities/availability/*` | BR-5-015/016/017/040, AC-23/26 | switch caller or delay retirement |
| **T5-81** | legacy writers | `$executeRawUnsafe` on restriction/availability tables outside `restriction-write.service.ts`; `crs.modifyReservation`/`crs.release`/`inventory.domain-service` writes after their gates; `INSERT INTO availability` | BR-5-018/021/025, INV-P5-12/21 | revert write; gate order violated ⇒ exit fails |
| **T5-82** | frontend business-rule duplication | duplicate availability payload declarations; client formulas computing sellable/available | BR-5-012/013, AC-20/21 | adopt shared type; delete math |
| **T5-83** | direct DB bypass / unvalidated writes | `body: any` on availability write routes; raw SQL added in Phase 5 outside sanctioned services; FK raw errors surfacing | BR-5-021(2), AC-28 | add DTO / route through service |
| **T5-84** | stale/undocumented flags | availability flags on platform DB system; `FEATURE_*` not in docs; code-level flag mutations (`process.env` writes, `featureFlag.set` in code) | BR-5-035/036, G-5, AC-37 | remove; docs restored |
| **T5-85** | dead migration paths | new files under `packages/db/migrations/**`; schema diffs | G-1, AC-44 | revert (hard stop) |
| **T5-86** | unsafe property fallbacks | `hotelId \|\| 'default'`, `'default'` literals in availability paths, bare-id availability mutations, cross-hotel aggregates | BR-5-032/049/031, G-7, AC-13 | revert to explicit rejection |

Implementation note: scans are authored as test-asserted scripts (Phase 4 precedent: grep gates T-31/T-39/T-59/T-65) so they run inside `pnpm test`, not as manual steps. Each scan lists its allowed-files exceptions explicitly (e.g., T5-81 allows `restriction-write.service.ts`).

---

## 27. Acceptance Criteria, Invariants, Migration-Rule Coverage

### 27.1 AC-01…AC-47 → tasks (nothing uncovered)

| AC | Anchor topic | Task(s) proving it |
|---|---|---|
| AC-01 | single truth across surfaces | T5-01, T5-30 (+ parity T5-10) |
| AC-02 | no second combination (overlay gone) | T5-40, scan T5-78 |
| AC-03 | arithmetic floors / sellLimit tighten-only | T5-01 |
| AC-04 | unresolved floors + UNKNOWN + nulls | T5-01, T5-04 |
| AC-05 | blocked vs unknown distinction | T5-01, T5-03 |
| AC-06 | actual zero = ELIGIBLE certainty-zero | T5-01, T5-03 |
| AC-07 | read failure ⇒ dimension UNRESOLVED | T5-08 |
| AC-08 | conflict ⇒ UNRESOLVED + sourceConflicts, no preference | T5-08 (+E-2 via T5-05) |
| AC-09 | absence ⇒ RESOLVED + provenance; unproven flagged | T5-08 |
| AC-10 | outcome RESOLVED iff all dimensions resolved | T5-08 |
| AC-11 | stop-sale display 0, counters untouched, order before remaining | T5-73, T5-20 |
| AC-12 | assertion rejection + 409 + no partial commit | T5-04 |
| AC-13 | context/default/route-property rejection (server+client) | T5-64, T5-68, T5-19 |
| AC-14 | hotel_id in every predicate; scoped authz equivalence | T5-65, T5-64, T5-66 |
| AC-15 | unscoped legacy retired/scoped | T5-22, scans T5-79/T5-80 |
| AC-16 | contract set = snapshot/matrix/recon only | T5-28, T5-78 |
| AC-17 | matrix shape preserved + interim label | T5-20, T5-30 |
| AC-18 | bed-type conservation law | T5-10, T5-20 |
| AC-19 | eligibility surfaced; UNKNOWN/BLOCKED distinct | T5-30, T5-31, T5-38 |
| AC-20 | no client availability math (11 formulas removed) | T5-30, T5-36, T5-77 |
| AC-21 | one shared contract type | T5-02, T5-82 |
| AC-22 | quick-book cells mapped + eligibility | T5-31, T5-38 |
| AC-23 | `GET /rates/availability` retired/scoped; no client call | T5-22, T5-32, T5-80 |
| AC-24 | no legacy gate (FO upgrade, quote signal) | T5-13, T5-14 |
| AC-25 | booking via create+assertion; engine/book gone | T5-24 |
| AC-26 | canonical logs/interval routes + payload + paging | T5-33, T5-80 |
| AC-27 | code-based error detection (server+client) | T5-26, T5-34 |
| AC-28 | typed validation before domain effect | T5-21, T5-83 |
| AC-29 | one route one owner (`/tax-rates`) | T5-27 |
| AC-30 | authority restriction write path; no raw writes post-cutover | T5-45, T5-46, T5-81 |
| AC-31 | CUTOFF unproven ⇒ excluded | T5-47, T5-87 |
| AC-32 | ordering honored; status quo until gates | T5-49, T5-50, T5-53 |
| AC-33 | deterministic write identity | T5-51, T5-52, T5-75 |
| AC-34 | delete semantics (reject-with-state; never releases) | T5-59, T5-60 |
| AC-35 | reconciliation read-only, flagged, authority wins | T5-57 |
| AC-36 | audit identity/append-only/no cascade-delete | T5-58 |
| AC-37 | single flag mechanism, defaults, docs, no code flips | T5-54, T5-55, T5-84 |
| AC-38 | flag-ON four-condition gate | T5-56 (evidence: T5-05/06/09/10) |
| AC-39 | cascade ON at deploy; wash blocked; sequence untouched | T5-54, T5-55 (exit report) |
| AC-40 | input validity (flagged/excluded, never trusted) | T5-05, T5-07, T5-08 |
| AC-41 | no phantom publication | T5-16 |
| AC-42 | KPI honesty (authority labels, declared occupancy, no mock-as-live) | T5-17, T5-35 |
| AC-43 | stage evidence (DB env, 0 harness-skip, baselines) | T5-76 + §34 battery (T5-70/72) |
| AC-44 | scope integrity (no schema, no deferral creep, hygiene order) | T5-62, §36 non-inputs, T5-53 checklist |
| AC-45 | affordance honesty (disabled-not-404 → wired) | T5-33, T5-48 |
| AC-46 | decision integrity (no reopening; contradictions documented) | §3 principle + T5-53 checklist + T5-63 |
| AC-47 | interim gate (flag OFF, no write-gating cutover, stub rollback) | T5-11, T5-55 |

**Coverage: 47/47.**

### 27.2 INV-P5-01…INV-P5-34 → tasks

| Invariant | Task(s) | | Invariant | Task(s) |
|---|---|---|---|---|
| INV-01 single authority | T5-78, T5-16, T5-44 | | INV-18 no client computation | T5-30, T5-36, T5-77 |
| INV-02 single combination | T5-40, T5-78 | | INV-19 eligibility surfaced | T5-30, T5-31, T5-38 |
| INV-03 read-only snapshot + port | T5-12, T5-71 | | INV-20 one shared type | T5-02, T5-82 |
| INV-04 fail-closed floors | T5-01, T5-04 | | INV-21 no legacy signal gates | T5-13, T5-14, T5-16, T5-78 |
| INV-05 UNKNOWN ≠ ELIGIBLE | T5-03, T5-30, T5-31 | | INV-22 delete never releases | T5-59, T5-60 |
| INV-06 nothing commits unresolved | T5-04, T5-75 | | INV-23 append-only movements | T5-58 |
| INV-07 arithmetic floors | T5-01 | | INV-24 one live assertion | T5-12 |
| INV-08 stop-sale semantics | T5-73, T5-20 | | INV-25 deterministic identity | T5-51, T5-52, T5-75 |
| INV-09 outcome integrity | T5-08 | | INV-26 ordering gates | T5-49, T5-50, T5-53 |
| INV-10 conflict ⇒ UNRESOLVED | T5-08 | | INV-27 flag integrity | T5-54, T5-55, T5-84 |
| INV-11 absence/provenance | T5-07, T5-08, T5-47 | | INV-28 reconciliation read-only | T5-57 |
| INV-12 single restriction writer | T5-45, T5-46, T5-81 | | INV-29 logs never authority | T5-61 |
| INV-13 hotel isolation | T5-64, T5-65, T5-79, T5-86 | | INV-30 no outbound publication | T5-16 |
| INV-14 context integrity | T5-64, T5-19 | | INV-31 room_inventory not a source | T5-44, T5-37 |
| INV-15 two sanctioned contracts | T5-28, T5-19 | | INV-32 deterministic error contract | T5-26, T5-34, T5-27 |
| INV-16 projection fidelity | T5-20 | | INV-33 freshness honesty | T5-29, T5-01 |
| INV-17 bed-type conservation | T5-10, T5-20 | | INV-34 state distinctions | T5-03, T5-30 |

**Coverage: 34/34.**

### 27.3 M-1…M-10 → tasks

| Rule | Treatment | Task(s) |
|---|---|---|
| M-1 sanctioned source or label | SPECIFIED | T5-29, T5-30, T5-33 (label interim) |
| M-2 write-gating re-point needs authority | SPECIFIED (gate) | T5-13, T5-14, T5-24 (gated) |
| M-3 read retirement respects E-5 | SPECIFIED | T5-22, T5-32, T5-88 |
| M-4 matrix migrates by value-sourcing | SPECIFIED | T5-20, T5-40, T5-10 |
| M-5 occupancy vs availability sourcing | SPECIFIED | T5-17, T5-35 |
| M-6 GBA views = server read models | SPECIFIED | T5-36 |
| M-7 admin/mobile greenfield on canonical contract | SPECIFIED (no current surfaces; guard via T5-28 review) | T5-28 |
| M-8 cannot-yet-migrate ⇒ declared, never broken | SPECIFIED | T5-33 (disabled-not-404), T5-30 (label) |
| M-9 no ordering inversion, no flag flips in code | SPECIFIED | T5-53, T5-84 |
| M-10 hygiene after replacement | SPECIFIED (gate) | T5-42, T5-43, T5-37 |

**Coverage: 10/10.**

---

## 28. Rollback / Safety

**Global safety frame:** no schema change ⇒ **no data migration to roll back**; legacy sources retained until Phase 11 ⇒ every code revert genuinely restores prior behavior; flags default OFF ⇒ previous behavior is the resting state (Phase 4 §11.7 precedent).

| Switch / change | Rollback mechanism | Restores | Proof |
|---|---|---|---|
| DI binding swap (T5-09) | revert one binding line | stub UNRESOLVED behavior | stub unit spec still green |
| `gba.a3.authoritative` ON (T5-56) | flag OFF | labeled legacy matrix view (decision-sanctioned) | T5-11/T5-55 default pins |
| AvailabilityPage cutover (T5-30) | revert commit | previous matrix-reading page | pre-migration fixtures |
| Quick-book mapping (T5-31) | revert | prior cells (known broken) | — |
| FO upgrade gate (T5-13) | revert `isAvailable` call | prior gate (legacy service retained) | unit |
| Quote signal (T5-14) | revert | prior `rate_restrictions` read | pricing-unchanged test |
| Route retirements (T5-22/23/24/25) | revert commit | route re-serves (post-migration callers = none ⇒ safe) | scans T5-80 |
| Restriction write service wiring (T5-46) | revert handlers | prior raw path (still correct until cutover) | handler tests |
| Affordance enable (T5-48) | disable affordance UI | disabled-not-404 state | AC-45 tests |
| Dual-write leg removal (T5-50) | revert (restore `:625`) | dual-write status quo (drift monitored) | T5-49 pin flips back |
| DTO validation (T5-21) | revert DTO | `any` accepts all (prior) | validation tests |
| Delete-reject (T5-59) | revert | DB-restrict error surfaces (still protected) | unit |
| Identity fixes (T5-52) | revert | undefined keys (prior) | idempotency tests |
| Hygiene removals (T5-42/43) | revert commit | files restored | build green |

**Safety rules:** (1) retire-then-revert ordering — callers migrate before routes disappear (T5-53 checklist); (2) no flag ever changes value inside a code commit (G-5 — flag flips are separate operational records); (3) any rollback executed later is recorded in exit evidence with its reason (FDS §24.7).

---

## 29. Execution Order

Derived strictly from §8 chains + gates. **Serial points are marked ‖.**

**Phase P0 — Ungated parallel build & evidence (start immediately):**
`T5-05` `T5-06` `T5-87` `T5-88` `T5-89` `T5-90` (evidence) ‖ `T5-01` `T5-02` `T5-03` `T5-04` (authority pins/types) ‖ `T5-12` `T5-15` `T5-16` `T5-18` (consumer verifies) ‖ `T5-19` `T5-20` `T5-21` `T5-26` `T5-28` (API test/hardening) ‖ `T5-29` `T5-34` `T5-38` (frontend build) ‖ `T5-45` `T5-47` `T5-48` (restriction path build) ‖ `T5-49` `T5-51` (audits/pins) ‖ `T5-54` `T5-55` (flags) ‖ `T5-57` `T5-58` `T5-60` `T5-61` (logs/recon) ‖ `T5-62` `T5-63` (schema gates) ‖ `T5-64`–`T5-68` (isolation) ‖ `T5-69`–`T5-75` (suites as tasks land) ‖ `T5-77`–`T5-86` (scan baselines). **First executed exit ⇒ `T5-76` (E-8).**

**Gate G1 — BLK-P5-01 closure (serial):**
`T5-05` ✓ + `T5-06` ✓ + `T5-07` (needs E-1) → `T5-08` ✓ → **`T5-09` (binding swap)** → `T5-10` (parity soak) ⇒ BLK-P5-01 evidence complete.

**Phase P1 — Post-G1 authority consumers (parallel):**
`T5-13` ‖ `T5-14` ‖ `T5-30` ‖ `T5-31` ‖ `T5-17` ‖ `T5-32` ‖ `T5-19` (flag-ON variants of T5-17/19 land only post-T5-56).

**Phase P2 — Read dispositions (respect E-5 timing):**
`T5-22` ‖ `T5-23` (after T5-14) ‖ `T5-32` (before/with T5-22) — all scanned by T5-80.

**Phase P3 — Restriction cutover (after T5-45/46 ready + G1):**
`T5-46` → `T5-39` → `T5-48` (+ `T5-33` wiring) — BLK-P5-02 closes here.

**Phase P4 — Booking retirement + write convergence (serial, highest risk B-4):**
`T5-24` (after P1+P2 reads disposed) → **`T5-50`** (after G1 + P2 + T5-49 pin satisfied + T5-53 checklist) → `T5-25` (writers gone) — BLK-P5-03 closes here. `T5-52` may run anytime after `T5-51`.

**Phase P5 — Cutover activations (operational, recorded):**
`T5-56` (flag ON — needs G1 evidence) → `T5-40` (overlay removal in ON window) → `T5-35`/`T5-37` completion.

**Phase P6 — Hygiene (last, G-12):**
`T5-42` → `T5-43` → doc chores → final exit battery (`T5-70`–`T5-76` re-run) → `T5-53` final checklist.

**Final validation gate:** full §34 battery green + all evidence records filed + blocker register status = closure-ready ⇒ Stage 5 entry (§35).

---

## 30. Parallelization Matrix

| Track | Contents | Safe together? | Must not overlap |
|---|---|---|---|
| **TR-1 Evidence** | T5-05, T5-06, T5-87…90 | yes, all read-only | nothing (fully parallel) |
| **TR-2 Authority pins/types** | T5-01…04, T5-02 | yes | — |
| **TR-3 API hardening** | T5-19, T5-20, T5-21, T5-26, T5-28 | yes | — |
| **TR-4 Restriction path build** | T5-45, T5-47 | yes with TR-2/3 | T5-46 (needs T5-45 review) |
| **TR-5 Frontend build (ungated)** | T5-29, T5-38, T5-34, T5-33 client conformance | yes | T5-30/31 (G1-gated) |
| **TR-6 Isolation + schema gates** | T5-62…68 | yes | — |
| **TR-7 Logs/recon** | T5-57, T5-58, T5-60, T5-61 | yes | T5-59 (touches delete path — coordinate with T5-12 tests) |
| **TR-8 Flags** | T5-54, T5-55 | yes | T5-56 (operational, post-G1) |
| **TR-9 Scans** | T5-77…86 | yes (baselines) | re-run after every phase |
| **Serial core** | G1 chain (T5-07→08→09→10) | **no parallelism inside** | everything gated on it |
| **Serial cutover** | T5-24 → T5-50 → T5-25 | **strictly serial** | any parallel retirement overlapping T5-50's gate check |
| **Hygiene** | T5-42/43 | parallel to each other, **after** all replacements | never before P1–P5 |

Rule: parallel tracks share no files (each track's file sets are disjoint above); where a file is shared (e.g., `availability-sales.controller.ts` in T5-21/46/39), tasks are sequenced inside their phase and rebase conflicts are resolved by order-of-landing recorded in T5-53.

---

## 31. Traceability Matrix (S3R → WS → tasks → verification → gate)

Every FDS §35 S3R ID maps to tasks; every task maps back to at least one S3R (§12–§26 `Why/Authority` lines). **Silently dropped: 0.**

### 31.1 Class A — S3R-001…051 (≡ BR-5-001…051)

| S3R | Rule | WS | Task(s) | Verification | Gate |
|---|---|---|---|---|---|
| 001 | Evaluator scope | C | T5-07, T5-08, T5-01 | evaluator unit + AC-07/08 | **BLK-P5-01 (E-1)** |
| 002 | Absence semantics | C | T5-08 | absence AC-09 tests | — |
| 003 | Conflict → UNRESOLVED | C | T5-08 (+T5-05 for E-2) | conflict AC-08 tests | E-2 deferred (default stands) |
| 004 | Outcome status | C | T5-08 | AC-10 tests | — |
| 005 | Fail-closed floors | C | T5-01, T5-04 | pin tests | — |
| 006 | Blocked/sellLimit | C | T5-01, T5-03 | pin tests | — |
| 007 | Stop sale | C/D | T5-73, T5-20 | GBA battery + projection | — |
| 008 | Single combination | D/G | T5-40, T5-78 | scan + parity | flag ON (T5-56) |
| 009 | Interim operational gate | I | T5-11, T5-55 | flag-default tests | — |
| 010 | Two sanctioned reads | B | T5-28 | route registry test | — |
| 011 | Eligibility surfacing | E | T5-30, T5-31, T5-38 | web tests | BLK-P5-01 for values |
| 012 | No client math / conservation | E | T5-30, T5-36, T5-77 | scan + component tests | — |
| 013 | Shared contract types | B/E | T5-02, T5-82 | typecheck + scan | — |
| 014 | Reconciliation read-only | J | T5-57 | AC-35 tests | — |
| 015 | `/rates/availability` retire | D/E | T5-22, T5-32, T5-80 | scans | **E-5 (timing)** |
| 016 | `/rates/engine` reads re-point | D | T5-14, T5-23 | AC-24 tests | — |
| 017 | `/rates/engine/book` retire | D | T5-24 | AC-25 tests | **BLK-P5-03** |
| 018 | Legacy counters | D | T5-25, T5-81 | scan | T5-24 order |
| 019 | FO upgrade gate | D | T5-13 | AC-24 tests | BLK-P5-01 |
| 020 | Error contract | B/E | T5-26, T5-34 (+T5-89) | AC-27 tests | — |
| 021 | Authority write path | F | T5-45, T5-46, T5-79 | AC-30 tests + scan | — |
| 022 | Restriction reads (authority) | F | T5-46 (removes `:384-405` raw reads) | AC-30 tests | — |
| 023 | A3 overlay removed | G | T5-40, T5-78 | scan | flag ON |
| 024 | Unproven stores not promoted | F | T5-47, T5-87 | guard test | **E-4** |
| 025 | Single write truth (modify) | D | T5-50, T5-25 | AC-32/33 tests | **BLK-P5-03** |
| 026 | No premature cuts | D | T5-49, T5-53 | pin test + checklist | — |
| 027 | Deterministic write identity | D | T5-51, T5-52, T5-75 | AC-33 tests | — |
| 028 | Availability-labelled figures | E | T5-17, T5-35 | AC-42 tests | BLK-P5-01 for authority source |
| 029 | Occupancy reporting | E | T5-17, T5-35 | AC-42 tests | — |
| 030 | No outbound publication | D | T5-16 | AC-41 tests | — |
| 031 | Hotel scoping (rules) | L | T5-64, T5-86 | AC-13/14 tests | — |
| 032 | Context rejection | L | T5-64, T5-19 | AC-13 tests | — |
| 033 | Executed evidence | M | T5-70, T5-72, T5-76 | AC-43 reconciliation | exit battery |
| 034 | Config declaration | I | T5-54 | doc check | — |
| 035 | Single flag mechanism | I | T5-55, T5-84 | AC-37 tests + scan | — |
| 036 | Flag documentation | I | T5-54 | doc check | — |
| 037 | Flag-ON four conditions | I | T5-56 (evidence T5-05/06/09/10) | AC-38 gate record | **BLK-P5-01** |
| 038 | Rollout sequence preserved | I | T5-54, T5-55 | AC-39 tests | — |
| 039 | Freshness honesty (LIVE_READ) | E | T5-29 | hook tests | — |
| 040 | Canonical API names | E | T5-33, T5-80 | AC-26 tests | — |
| 041 | No silent-404 affordance | E/G | T5-33, T5-48 | AC-45 tests | BLK-P5-02 |
| 042 | Delete never releases | J | T5-60 | AC-34 tests | — |
| 043 | Delete with state rejected | J | T5-59 | AC-34 tests | — |
| 044 | One route one owner | B | T5-27 (+T5-90) | AC-29 test | **E-7** |
| 045 | `room_inventory` legacy | G/E | T5-44, T5-37 | scan + component tests | — |
| 046 | Input validity | C | T5-05, T5-07, T5-08 | AC-40 tests | **E-1 (part)** |
| 047 | Context scoping (API) | L | T5-64 | AC-13 tests | — |
| 048 | Scoped authz equivalence | L | T5-66 | per-controller authz tests | — |
| 049 | Raw SQL hotel-scoping | L | T5-65, T5-79, T5-86 | scan + scoping tests | — |
| 050 | No legacy signal gates/publishes | D/E | T5-13, T5-14, T5-31, T5-78 | AC-24/25 tests | BLK-P5-01/03 |
| 051 | Retained counts hotel-scoped | D | T5-22, T5-79 | scan | **E-5 (timing)** |

### 31.2 Class B — S3R-052…057 (BLK-P5-01…06)

| S3R | Blocker | Task(s) | Gate |
|---|---|---|---|
| 052 | BLK-P5-01 evaluator + evidence | T5-05, T5-06, T5-07→08→09→10 | **hard: E-1+E-3 recorded** |
| 053 | BLK-P5-02 affordances | T5-45, T5-46, T5-48, T5-33 | DS-04 |
| 054 | BLK-P5-03 ordering | T5-49, T5-53 (checklist), phases P1–P4 in §29 | DS-01→DS-03→DS-05 verbatim |
| 055 | BLK-P5-04 suite evidence | T5-70…T5-76 | exit battery |
| 056 | BLK-P5-05 flag inventory | T5-54, T5-55 | AC-37/39 |
| 057 | BLK-P5-06 scope exclusions | §36 non-inputs; no task implements it | explicit deferral authority |

### 31.3 Class C — S3R-058…065 (E-1…E-8)

| S3R | Evidence | Task(s) | Effect |
|---|---|---|---|
| 058 | E-1 population audit | T5-05 | BLK-P5-01 (hard) |
| 059 | E-2 precedence intent | T5-05 (collected with E-1) | default stands (UNRESOLVED) if none |
| 060 | E-3 DI runtime confirmation | T5-06 | BLK-P5-01 (hard) |
| 061 | E-4 CUTOFF meaning | T5-87 | BR-5-024 store stays out |
| 062 | E-5 external consumers | T5-88 | retirement **timing only** |
| 063 | E-6 OTA branch | T5-89 | informative |
| 064 | E-7 `/tax-rates` winner | T5-90 | gates T5-27 |
| 065 | E-8 baseline reconciliation | T5-76 | first executed exit = Stage 5 entry |

### 31.4 Class D — S3R-066…077 (G-1…G-12)

| S3R | Guardrail | Task(s) enforcing |
|---|---|---|
| 066 | G-1 no schema/migrations | T5-62 (every exit) |
| 067 | G-2 no tests deleted/weakened | §34 battery rules (T5-70/72/76) |
| 068 | G-3 no new dependencies | §36 non-inputs; diff review at T5-53 |
| 069 | G-4 legacy retained till Phase 11 | T5-22/23/25 dispositions (retire **routes/callers**, keep table/impl) |
| 070 | G-5 no code-level flag changes | T5-55, T5-84 |
| 071 | G-6 no DB-touching impl here | T5-57 (read-only), T5-62 |
| 072 | G-7 hotel scoping | T5-64…T5-67, T5-79, T5-86 |
| 073 | G-8 one authority, one combo | T5-78, T5-40 |
| 074 | G-9 no partial scope creep | §36 non-goals + §32 carry-over rule (F-24) |
| 075 | G-10 decision preservation | §3 principle; T5-53 checklist "no reopened decision" |
| 076 | G-11 evidence never fabricated | T5-05/06/87–90/76 read-only evidence tasks |
| 077 | G-12 hygiene last | T5-42, T5-43 gated by §29 P6 |

### 31.5 Class E — S3R-078…089 (DS-01…DS-12)

| S3R | Entry condition | Plan realization |
|---|---|---|
| 078 | DS-01 evaluator drafted + E-1/E-3 scheduled | §13 contract (T5-07/08) + T5-05/T5-06 |
| 079 | DS-02 contract types + surfacing | T5-02, T5-29, T5-30, T5-31 |
| 080 | DS-03 E-5 before retirement schedule | T5-88 → T5-22/23/24/25 timings |
| 081 | DS-04 write path + E-4 before CUTOFF | T5-45, T5-46, T5-47, T5-87 |
| 082 | DS-05 ordering verbatim | §8 CHAIN-5 + §29 P4 + T5-49/T5-53 |
| 083 | DS-06 display rules (analytics deferred P-20) | T5-17, T5-35; §33 defers contract |
| 084 | DS-07 out-of-scope carry (P-19) | §33; T5-16 guard (no publication) |
| 085 | DS-08 out-of-scope carry (TR-14.3) | §33 |
| 086 | DS-09 E-8 first executed exit | T5-76 |
| 087 | DS-10 config documentation | T5-54 |
| 088 | DS-11 canonical contracts + E-7 | T5-33, T5-21, T5-27 (+T5-90) |
| 089 | DS-12 E-1 population audit | T5-05 |

### 31.6 Class F — S3R-090…116 (F-01…F-27)

| S3R | Finding | Task(s) | | S3R | Finding | Task(s) |
|---|---|---|---|---|---|---|
| 090 | F-01 evaluator | T5-05, T5-06, T5-07, T5-08, T5-09, T5-10 | | 104 | F-15 channel log non-authority | §33 (P-19) + T5-61 guard |
| 091 | F-02 snapshot consumers | T5-29, T5-30, T5-35 | | 105 | F-16 identity/`default` | T5-51, T5-52, T5-86 |
| 092 | F-03 restriction ownership/overlay | T5-40, T5-46, T5-45 | | 106 | F-17 env/zero-skip | T5-54, T5-70, T5-76 |
| 093 | F-04 dual-write drift | T5-49, T5-50, T5-57 | | 107 | F-18 flag documentation | T5-54, T5-55 |
| 094 | F-05 FO upgrade gate | T5-13 | | 108 | F-19 route-property divergence | T5-64, T5-66 |
| 095 | F-06 `/rates/availability` | T5-22, T5-79, T5-80 | | 109 | F-20 `/tax-rates` duplicate | T5-27, T5-90 |
| 096 | F-07 book gate | T5-24, T5-31 | | 110 | F-21 dead artifacts | T5-42, T5-43 |
| 097 | F-08 canonical routes | T5-33, T5-21 | | 111 | F-22 stale baselines | T5-76 |
| 098 | F-09 client math/bed types | T5-30, T5-36, T5-77 | | 112 | F-23 doc drift | T5-63, §34 doc-consistency |
| 099 | F-10 quick-book mapping | T5-31, T5-38 | | 113 | F-24 Phase 4 carry-overs | §32 (owned by Phase 4) |
| 100 | F-11 legacy availability figures | T5-32, T5-37, T5-44 | | 114 | F-25 duplicate availability routes | T5-27, T5-43 |
| 101 | F-12 two screens disagree | T5-10, T5-30 | | 115 | F-26 raw SQL | T5-45, T5-65, T5-79, T5-83 |
| 102 | F-13 duplicate contract types | T5-02, T5-82 | | 116 | F-27 substring detection | T5-26, T5-34, T5-89 |
| 103 | F-14 cache freshness | T5-29 (+BR-5-039 pin) | | | | |

### 31.7 Cross-cutting groups — S3R-G1…G6

| S3R-G | Content | Realized by |
|---|---|---|
| G1 | §8.3 interim rules (flag OFF, structural consumption, no write-gating re-point, stub kept) | T5-11, T5-29, T5-33, T5-55; §28 rollback (stub revert) |
| G2 | §9.2 bed-type partition + conservation | T5-10, T5-20 |
| G3 | §10.2 endpoint dispositions (8 rows) | T5-22…T5-25, T5-13, T5-32, T5-44 |
| G4 | §19.1 input-ownership (11 stores) | T5-05, T5-07, T5-47 |
| G5 | §22 ordering + §28 boundary (B-1 before B-2…B-6; B-4 highest risk) | §8 chains, §29 phases, T5-49/T5-53 |
| G6 | §25/§26/§27 duty lists (isolation 5, frontend 8, API 8) | WS-L (T5-64…68), WS-E (T5-29…38), WS-B (T5-19…28) |

**S3R coverage: 116/116 + 6/6 groups. Tasks referenced: 90/90 (T5-01…T5-90).**

## 32. Phase 4 Carry-over Dependencies (non-actions — F-24/S3R-113)

Phase 5 **may not assume these are done** and **may not execute them** (they remain Phase 4-owned; `17_PHASE4_READINESS_REVIEW.md:92-99`):

| Carry-over | Status in Phase 5 | Handling |
|---|---|---|
| **BLK-1 / Deviation A** — `GUARANTEED_BLOCK` wash exclusion not shipped (t45 spec skipped at `:293`) | open | wash flag stays blocked (T5-54 documents; T5-76 records skip); no Phase 5 task activates wash |
| **BLK-2 / Deviation B** — durable wash/release/attrition store undecided | open | same |
| **Deviation C** — no schema changes (Phase 2 DB foundation skipped) | absorbing | reinforces G-1 → T5-62 |
| **Deviation D / NB-1…NB-6** — doc chores | absorbed as **Stage 4/6 doc duties only** | doc-consistency sweep in §34 (F-23); no code effects |
| **T-55** — Phase 4 test-maintenance task | Phase 4-owned | not re-planned here; baselines reconciled as-is (T5-76) |
| **Cascade flag deploy configuration** (`gba.consumers.cascade` ON at deploy) | documented, not executed in code | T5-54 documents; deployment action is operational |
| **Canonical cutover** (Phase 4 sequencing leftovers) | not executed | dispositions executed only via §29 phases under BLK-P5-03 |
| **Open actions list / `17_:70` deviation line** | evidence only | cited in §6 baselines |

Verification: §34 exit checklist includes "no carry-over from this table was executed under a Phase 5 heading" (AC-44 half, G-9).

---

## 33. Explicitly Deferred (nothing silently dropped)

| Deferred item | Authority | Notes |
|---|---|---|
| Availability analytics contract (canonical occupancy formula, real-time vs batch) | P-20, DS-06, BR-5-029 note | display rules remain; contract not started |
| Channel/push outbound availability publication (`channel_availability_log` semantics) | P-19, DS-07, BR-5-030 | T5-16 guard only (never publish) |
| Admin panel + mobile availability surfaces | P-19, audit §5.7 | zero current surfaces; M-7 greenfield rule recorded (T5-28 review) |
| `GET /intervals/audit` business-owned route replacement | TR-14.3, DS-08 | non-authoritative log marked as such (S3R-104 guard) |
| Legacy `availability` table deletion / migration / backfill | P-16, G-1, Phase 11 | read-side retirement only |
| `FEATURE_GBA_WASH_SCHEDULER_ENABLED` activation; `canonicalRead`→`canonicalWrite` sequence progression | P-21, Deviations A/B | preserved untouched (T5-54 documents) |
| E-2 precedence ranking (if conflicts later evidenced) | DS-01.3 default | conflict → UNRESOLVED stands until ratified decision (separate stage) |
| Future availability cache (TR-10.3 invalidation) | BR-5-039, F-14 | "no cache" documented in T5-29; nothing implemented |
| `BLK-P5-06` scope exclusions | P-19, P-20, TR-14.3 (explicit authority) | never partially implemented under another heading |
| Transfer/deposit/waitlist semantics | ADR-072 (Reservations roadmap) | unrelated to availability authority |

---

## 34. Stage 6 Quality Gates (executed in Stage 6, defined here)

Exit battery (run in order; all green + evidence filed = Stage 6 exit):

1. **G-1 schema gate** — T5-62 assertions (`git status packages/db` clean; `prisma validate` ×2; `migrate diff` empty; migration count baseline unchanged).
2. **Type/lint** — `cd apps/api && pnpm typecheck`; `cd apps/web && pnpm typecheck` (0 errors) + `pnpm lint` (exit 0); API `lint` is a no-op (documented, not relied on).
3. **Unit + contract suites** — `cd apps/api && pnpm test` with `AVAILABILITY_TEST_DATABASE_URL` set; 0 skips attributable to missing env; includes WS-N scan tests (T5-77…86).
4. **Frontend suite** — `cd apps/web && pnpm test` (+ typecheck/lint above); availability coverage > 0.
5. **GBA regression** — GBA battery green (79-suite baseline preserved).
6. **Baseline reconciliation (E-8)** — T5-76 table: API 178/1470/1skip/6fail (known 2-suite/6-test failures + `t45` skip), web 1-suite/10-test baseline, deviations **explained, never redefined**.
7. **Evidence register check** — E-1, E-3 (or explicit blocked status), E-4, E-5, E-6, E-7, E-8 records present; E-2 documented as deferred-default.
8. **Blocker register check** — BLK-P5-01…05 closure evidence or explicit "still blocked" status; no gate silently skipped.
9. **Flag report** — seven flags' values + gate evidence for any ON; zero code-level flag changes in diff (T5-84).
10. **Ordering checklist (T5-53)** — DS-01→DS-03→DS-05 honored; no step skipped; no reopened decision (G-10); no carry-over executed (§32); no out-of-scope item implemented (G-9).
11. **Doc consistency (F-23)** — cited anchors re-checked (T5-63); NB-1…NB-6 doc chores state; drift corrected in docs only.
12. **Rollback readiness** — §28 mechanisms verified intact (stub present, flag OFF default, legacy retained).

Non-negotiable: no exit passes with a redefined baseline, fabricated evidence, or a mutated decision record.

---

## 35. Stage 5 Readiness Inputs (what Stage 5 must receive)

Stage 5 (implementation) **starts only** with this plan + the following verified:

1. `04_IMPLEMENTATION_PLAN.md` complete: §1–§37, 90 tasks, all S3R/AC/INV/M cross-references resolve.
2. Stage 1–3 artifacts closed and unchanged (`01`/`02`/`03` byte-stable).
3. Evidence status known at handoff: E-1/E-3 state recorded (in-progress vs recorded vs blocked); E-4…E-7 tasks opened (T5-87…90 issued as scheduled work).
4. Baselines frozen exactly as §6 (178/1470/1skip/6fail; web 1-suite/10-test; GBA 79/766/1skip/0fail; 51+2 migrations; HEAD reference).
5. Gate tasks defined: T5-05/06/87/88/89/90/76 present with their read-only evidence contracts.
6. §29 execution order + §30 parallelization acknowledged (serial points: G1 chain, P4 cutover, P6 hygiene).
7. G-1 constraint confirmed (zero schema diff at handoff — T5-62 first run).
8. Stage 5/6 not started during Stage 4 (verified by git status: only `04_IMPLEMENTATION_PLAN.md` added under `phase-5/`).

Explicit: Stage 5 is **not** begun by this document; §10 of this plan schedules Stage 5's entry criteria, nothing more.

---

## 36. Non-Goals / Scope Fence (input to Stage 5's §27-style review)

**Not inputs, not to be created/modified during Stages 5–6:**

1. Database schema, migrations, indexes, constraints, seeds, backfills, generated clients (`packages/db/schema.prisma`, `packages/db/migrations/**`, Prisma generate) — G-1, AC-44.
2. GitHub/remote state; no fetch/pull/restore/reset/checkout/cherry-pick; no history rewrites.
3. `apps/admin`, `apps/mobile` availability features (zero current surfaces; M-7 only if greenfield requested later).
4. Reservations rebuild phases, ADR-072 transfer work, Activities/Events domain boundaries (ADR-074) — availability touches reservation writes only via existing assertion ports.
5. New dependencies/lockfile changes (G-3); platform feature-flag system changes (G-5; availability flags stay env `FEATURE_*`).
6. Legacy `availability` table lifecycle (P-16 — retained read-only until Phase 11).
7. Wash/release/attrition activation; `canonicalRead`/`canonicalWrite` progression (P-21, Deviations A/B).
8. Outbound channel publication; analytics/occupancy contract (P-19/P-20).
9. UI/UX redesign beyond surfacing duties (AC-19/45, M-8) — no layout/copy/branding changes.
10. Any new business rule: if a required rule is absent, the task **stops and escalates** (Stage 3 §36 rule) — never invent.
11. Test weakening/deletion (G-2), baseline redefinition, evidence fabrication (G-11).
12. Phase 4-owned carry-overs (§32 table).

---

## 37. Final Verdict

**Stage 4 complete.** Artifact completeness / Phase-4 parity gate: **STATE B — COMPLETE WITH EMBEDDED COVERAGE** (20/20 layers cited; no missing-artifact State C condition; no fabricated fill). Stage 3 validation: **PASSED** (47/47 ACs, 34/34 INV, 51/51 BR rows, 116/116 S3R + 6 groups, statuses 88/10/16/2, 0 dropped, 0 reopened).

Plan totals: **14 workstreams (WS-A…WS-N)**, **90 tasks (T5-01…T5-90)** covering 47 AC + 34 INV + 10 M + 116 S3R + 6 groups; 7 dependency chains; 6 phases (P0–P6) + 1 serial gate (G1); hard blockers open: **BLK-P5-01 (E-1+E-3), BLK-P5-02 (DS-04), BLK-P5-03 (ordering)**; evidence tasks E-4…E-7 issued (T5-87…90); E-8 = first executed exit (T5-76).

Safety: schema impact **zero**; DB touched only by tests; flag values unchanged; legacy retained to Phase 11; rollback = revert + flag OFF (§28).

**Stage 5 not started. Stage 6 not started.**

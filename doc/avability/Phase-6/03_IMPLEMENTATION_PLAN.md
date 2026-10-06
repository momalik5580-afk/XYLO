# XYLO Availability Phase 6 — Implementation Plan

**Artifact:** `docs/availability/phase-6/03_IMPLEMENTATION_PLAN.md`
**Phase:** 6 — Reconciliation Closure, Population Cutover, Legacy Counter Retirement, Controlled Canonical Cutover
**Mode:** **PLANNING ONLY.** No code, schema, migration, DB data, test execution, reconciliation run, cutover, flag flip, legacy deletion, or legacy writer retirement is performed or authorised by the act of authoring this document.
**Convention authority:** `02B_OWNER_DECISION_RESOLUTION.md` Decision 4 (OI-10 / TD-6-15) — **Option B**: `03_IMPLEMENTATION_PLAN.md` + `04_EXECUTION_EVIDENCE.md`; tasks `T6-nn`; evidence `E6-nn`; **no new gate namespace**; **no stage model**.
**Evidence register:** `04_EXECUTION_EVIDENCE.md` is authorised by OI-10 and is **not created here**. It is created when execution begins. Section 14 defines the `E6-nn` entries it must hold.

**Standing prohibitions (binding on this document and on every task in it):**

1. No production code, schema, migration, DB write, test run, flag flip, reconciliation execution, cutover execution, legacy deletion, or legacy writer retirement occurs because this plan exists. Each is a separately gated task below.
2. No new business rule is created here. Every task cites an existing `BD-6-*` / `TD-6-*` / `CO-6-*` / `DS-*` / `E-*` authority or an explicit ratification from the Phase 3/4/5 corpus.
3. No Phase 5 document, task ID, or status is reopened, retro-closed, amended, or rewritten. Phase 5 remains CLOSED at 12/12 exit gates.
4. No document in `docs/availability/phase-4/`, `phase-5/`, or `docs/enterprise/` is modified by this plan. Annotation of a closed document is itself a gated task (T6-07, T6-08) and is performed under `annotate, never rewrite` (TD-6-07, L-r-25).
5. No `T5-*`, `BLK-P5-*`, `E-1…E-8`, or `S3R-nnn` identifier is reused as a Phase 6 task, gate, or evidence ID (`01` §C.4 D-6). Phase 5 evidence IDs `E-1…E-8` are referenced **as dependencies only**, never renumbered or reissued.
6. No `OI-*` identifier is used as a task, gate, or evidence ID. `OI-*` are `02A`-local and carry no task authority (02B Decision 4).
7. Nothing enters this plan without an authoritative decision or disposition (`02` §R). Anything discovered in source that no decision covers is recorded as an out-of-plan observation (§4.4), not added as a task.
8. No stage is invented. The only sequencing expressed is the ratified `DS-01 → DS-03 → DS-05` and `canonicalRead → soak → canonicalWrite`.

---

## 1. Purpose and Scope

### 1.1 Purpose

Translate the completed Phase 6 decision chain — `01_FORENSIC_AUDIT.md` → `02_DECISION_RESOLUTION_ANALYSIS.md` → `02A_OPEN_DECISION_EVIDENCE_RESOLUTION.md` → `02B_OWNER_DECISION_RESOLUTION.md` — into an executable, dependency-ordered task register with explicit verification, evidence, stop, and rollback conditions.

This plan **implements decisions that already exist.** It does not create, reinterpret, or silently resolve any undecided matter. Where a sequencing reading is required to execute an already-final decision, that reading is written down explicitly and labelled as an ordering interpretation, never as a decision (the readings are set out in §8.4 and registered, with their escalation triggers, in §17).

### 1.2 Written scope of record

Phase 6's scope is quoted, not reinterpreted (`01` §B, `02` §B.2):

| Ref | Statement | Source |
|---|---|---|
| B-1 | "Cutover of a Reservation population to assertion authority, deterministic population migration at scale, legacy counter retirement, dual-source reconciliation closure, and removal of `crs-engine` inventory calls." | `docs/enterprise/availability-phase3-implementation-plan.md:427` (§13.4) |
| B-2 | "Deferred to Phase 6 \| population cutover, legacy counter retirement, dual-source reconciliation." | same file `:442` |
| B-3 | Legacy `availability`/`inventory` counter retirement → Phase 6; population cutover at scale; dual-source reconciliation closure → Phase 6; "Removing `crs-engine` inventory calls from untouched legacy paths" → Phase 6 | same file `:644-646` |
| B-4 | "Reconciliation, population cutover, and legacy retirement belong to Phase 6; this specification does not perform them." | `availability-phase3-domain-specification.md:445` |
| B-5 | "Legacy counter cutover — RESOLVED — deferred to Phase 6" (D-20) | `availability-phase3-business-rules-decision-sheet.md:22` |

Operationalised as the seven written scope elements named in the authoring brief: **reconciliation closure · population cutover · legacy counter retirement · removal of the remaining `crs-engine` inventory calls · applicable legacy disposition/retirement work · controlled canonical cutover · evidence required to prove completion.**

### 1.3 Enumerated plan scope

`02` §R fixes scope, and `02A` §L.3 ground 2 confirms it is *closed*:

> TD-6-01 + TD-6-10 (one bundled task), TD-6-03 (scheduled report-only reconciler + I-2 comparator), TD-6-04 (census then first-touch tooling under D-21), BD-6-06 / T5-59 (typed delete-reject), TD-6-09 (conditional, both branches specified — **now decided: Option A** by 02B Decision 3), plus execution of BD-6-04 and BD-6-05 once their gates are set. **Nothing else may enter the plan without a decision ID.**

The four owner decisions finalized in `02B` add their own plan-scoped execution units, each explicitly named by `02B` as *"what remains gated for implementation"*:

| Owner decision | Execution unit named by `02B` | Tasks here |
|---|---|---|
| OI-03 / BD-6-03 (gate-based EOL) | "the retirement task itself (route removal → 404 + writer scan proving no residual caller), plus `G-2` `InventoryDomainService` removal and `TD-6-02` execution" | T6-16 … T6-19 |
| OI-04 / BD-6-05 (soak criteria) | "opening the soak window (two operational flag actions + gate evidence), running it, and recording SO-1…SO-4" | T6-21 … T6-25 |
| OI-07 / TD-6-09 (Option A read contract) | "the read-contract change itself (additive field + calculator combination + FDS §11.3 annotation note + three DB-backed tests), and the subsequent re-baselining of `/reconciliation`" | T6-09, T6-10 |
| OI-10 / TD-6-15 (convention) | "authoring `03_IMPLEMENTATION_PLAN.md`" | this document |

### 1.4 Boundary — what is deliberately absent

See §4. Nothing in §1.2/§1.3 is redesigned, widened, or narrowed here.

---

## 2. Authoritative Inputs

### 2.1 Authority hierarchy (mandatory)

| Tier | Source | Role |
|---|---|---|
| **A** | Final decisions in `02B_OWNER_DECISION_RESOLUTION.md` | Final for OI-03, OI-04, OI-07, OI-10 |
| **B** | Resolutions in `02A_OPEN_DECISION_EVIDENCE_RESOLUTION.md` | Status of all 30 open items; §H blocker assessment; §I execution gates; §K.1 two-tier baseline rule; §L.3 conditions C1–C5 |
| **C** | Decision dispositions in `02_DECISION_RESOLUTION_ANALYSIS.md` | `BD-6-*`, `TD-6-*`, `CO-6-*`, §G reconciliation, §H cutover, §I dispositions, §L `L-r-01…25`, §M `M-r-*`, §P invariants |
| **D** | Forensic facts in `01_FORENSIC_AUDIT.md` | As-built topology, read/write surfaces, `G-1…G-16`, `H-1…H-7`, `I-1…I-7`, `J-1…J-7`, `N-1…N-6` |
| **E** | Ratified upstream Phase 3/4/5 rules cited by A–D | FDS §11.3/§24/§26/§28/§31, `P-*`, `BR-5-*`, `REQ-25.*`, `INV-*`, `AC-*`, `D-21` |
| **F** | **Current local source, schema and configuration** | Exact file/line/symbol targeting for every task |

**Conflict rule.** If tier F conflicts with tier A–E, **preserve the finalized decision and document the discrepancy** — never silently change the decision. Three such instances were found during targeting and are documented in place: §7 (T6-16 RC-4 ordering), §9.4 (I-2 register wording), §11.2 (SO-1 status vocabulary).

**Workspace rule.** The local tree is authority. No fetch, pull, restore, reset, or replacement of local state from GitHub is performed or required (`01` §P.1 P-9; `02` §0.3).

### 2.2 Inputs read in full for this plan

| Document | Bytes | Role |
|---|---|---|
| `01_FORENSIC_AUDIT.md` | 60,728 | Finding baseline |
| `02_DECISION_RESOLUTION_ANALYSIS.md` | 95,522 | Disposition baseline |
| `02A_OPEN_DECISION_EVIDENCE_RESOLUTION.md` | 59,549 | Status + gate baseline |
| `02B_OWNER_DECISION_RESOLUTION.md` | 48,880 | Final owner decisions |

### 2.3 Local workspace targets verified while authoring

Verified by direct read in this session (tier F), not inherited from any summary:

| Target | Verified state |
|---|---|
| `POST engine/modify` route | `rates-inventory.controller.ts:6` `@Controller('rates')`, `:108` `@Post('engine/modify')`; sole in-repo occurrence outside tests |
| In-repo consumers of `engine/modify` | **∅** — only the route definition plus three retirement specs that pin its existence (`t523`, `t524`, `t525`) |
| `crs.modifyReservation` | `crs-engine.service.ts:290`; legacy gate `:364`; legacy counter writes `:391-402`; `tx.reservations.update` `:411`; `reservation_changes` `:425` |
| `assertion` occurrences in `crs-engine.service.ts` | **zero** |
| `InventoryDomainService` | `inventory.domain-service.ts:56 checkAvailability`, `:160 reserve`, `:177 release`, `:190/:198/:209/:238/:267` dead methods; raw `UPDATE availability` at `:182,:228,:257,:276` |
| Matrix defect anchors | `availability-sales.controller.ts:305-307` filter construction, `:306` `rtFilterRt` (`rt` alias), `:326` unconditional query, `:333` interpolation, `:349`/`:400` secondary `rtFilter` misuse, `:377`/`:390` direct `roomType` embedding; route `:290` |
| Authority read surface | `availability.controller.ts:9` `@Controller('properties/:propertyId/availability')`, `:13 snapshot`, `:21 reconciliation` |
| Snapshot combination | `snapshot-calculator.ts:14` — three terms, no balance term |
| Read omission | `reservation-consumption.adapter.ts:20-22` excludes `population: 'ASSERTION_MANAGED'`; `availability-source.adapter.ts:140` passes quantity through; **no reader of `availability_assertion_balances` in the read path** |
| Write gate (correct) | `availability-assertion.service.ts:894 capacityRejection`, `:900` `remaining = day.sellableAvailable − balances.get(...)` |
| First-touch assignment | `availability-assertion.service.ts:272-273`, `:552`, `:563-570` |
| Reconciliation service | `availability-reconciliation.service.ts:18` `availability.findMany`, `:22` `expected`, `:23` `observed`, `:24` severity, `:32` `automaticRepair: false`; **no `@Cron` anywhere under `modules/availability`** |
| GBA `LEGACY_DRIFT` | `gba-reconciliation.service.ts:131` `@Cron(EVERY_HOUR)`, `:162` flag-independent on-demand `runReconciliation(hotelId)`, `:347` `run('LEGACY_DRIFT', …)`, `:379` exact-match `if (left !== right)`, `:39` `GbaDetectorStatus = 'CLEAN' \| 'FLAGGED' \| 'PENDING' \| 'ERROR'`, `:193-200` status derivation, `:403` `automaticRepair: false` |
| Pickup read/write gates | `allotment.controller.ts:302` comment, `:313` `canonicalRead` read; `reservation-pickup-cascade.service.ts:94` comment, `:102` `canonicalWrite` → `legacyReadOnly` |
| Cascade gate | `events.consumer.ts:150`, `:171` `gba.consumers.cascade` |
| Module bindings | `availability.module.ts:45` `RESERVATION_AVAILABILITY_PORT → AvailabilityAssertionService`, `:50` `UnresolvedRestrictionAdapter` retained, `:51` `RESTRICTION_EVALUATOR → PrismaRestrictionAdapter` |
| Delete path | `reservation.repository.ts:878` `async delete(id)`, **no availability-state check**; `:89` `inventoryDomain` injected but never invoked; `availability-error-codes.ts:21` `RESERVATION_HAS_AVAILABILITY_STATE: 409` defined, never thrown |
| Flag state | `.env:68` `FEATURE_GBA_A3_AUTHORITATIVE=true` (**only ON flag**); `.env.example:100-106` all seven `false`, `:101` deploy invariant, `:105-106` sequencing comment |
| Schema | `schema.prisma:433 model availability` (`reserved` `:439`, `available` `:441`, unique `[hotel_id, date, room_type]`); `:17316 model availability_assertion_balances` (`asserted_quantity`, PK `[hotel_id, room_type, stay_date]`); `:17391 model reservation_availability_state` (`population String @db.VarChar(30)`, index `[hotel_id, population]`); `:10413 model room_inventory`; `:137 model allotment_pickup` |
| G-14 / G-15 / G-16 surfaces | `ws-n-scan-gates.test.ts:118-121` `CLIENT_MATH_ALLOWED`; `rates-inventory/page.tsx:140`, `use-crs-book.ts:91`, `front-office.api.ts:193` retired-route delay entries; `ws-n-scan-gates.spec.ts` pins the dead repository DI |
| Test assets | `apps/api` jest regex `.*\.spec\.ts$`; DB-backed convention exists (`*.postgres.spec.ts`); `t557-reconciliation-readonly.spec.ts`, `t561-logs-never-authority.spec.ts`, `t528-fixed-surface-gate.spec.ts`, `t519-authority-surface.spec.ts` are the operative pins; **api `lint` script is `echo 'ok'`** |

---

## 3. Phase 6 Invariants

Carried verbatim from `02` §P (L-r register) and `02` §G.2. These are **non-negotiable**; every task is executed under them. Full citations at `02` §L.

### 3.1 Authority and truth

1. One sellable number, produced only by Availability (`L-r-01`, `L-r-02` — P-12, P-11).
2. Exactly four fact kinds; provenance and `UNRESOLVED` always surfaced (P-11, P-13).
3. No derived or relabelled second computation (P-17).
4. **Legacy never wins a divergence** (`L-r-09` — REQ-25.2.2).

### 3.2 Reconciliation

5. **Report-only. `automaticRepair` is always `false`.** Balances are never "fixed" (`L-r-08`, `L-r-21` — REQ-25.2.1/.4).
6. Reconciliation output **never** feeds availability decisions, eligibility, or publication (`L-r-10` — REQ-25.3).
7. Reconciliation output lives only in the FDS §26.2 *reconciliation evidence* layer (`L-r-10`, §G.2 inv. 5).
8. Only one sanctioned legacy-vs-authority comparison exists (`REQ-25.1`); an additional comparator must be introduced as a further **sanctioned, read-only** comparison — never as a second truth (`02` §G.2 inv. 4).
9. Availability-state rows are written only through the assertion port (FDS §26.3, `L-r-21`).
10. **No auto-repair under any flag, ever** (`02` §G.2 inv. 7).

### 3.3 Cutover and writes

11. **`DS-01 → DS-03 → DS-05`; no step skipped** (`L-r-11` — FDS §24.2).
12. **No premature cuts** while the authority is not operational: no partial cuts, no "temporary" removal of a legacy leg, no flag-ON experiments with the stub bound (`L-r-12` — BR-5-026, FDS §24.3).
13. The legacy counter writer ceases **only** when the last CRS booking/modify path is re-pointed (`L-r-13` — BR-5-018/025, FDS §24.4).
14. First-touch population strategy; **no bulk backfill**; `LEGACY` refuses assertion fail-closed (`L-r-22` — D-21).
15. Deterministic write identity on every availability-bearing write (`L-r-14` — BR-5-027).
16. Movements append-only; balances = Σ movements (`L-r-21`).

### 3.4 Flags and rollback

17. Flags default OFF; **no flag is changed inside a code commit**; flag changes carry recorded gate evidence (`L-r-05`, `L-r-16` — P-21, G-5, FDS §24.8.4 / §28.2).
18. **`canonicalRead` → soak → `canonicalWrite`; the order is never altered; never both ON** (`CO-6-05` — FDS §28.3/§28.5, P-21).
19. Rollback = flag OFF or one binding line; **no schema change required** (`L-r-15` — FDS §24.7).
20. Flag posture starts from the **as-is census**, never from the documented end-state (`01` J-AUD-01, `02A` §B.3).

### 3.5 Scope discipline

21. **No schema change, no migration authoring, no backfill** (`L-r-23` — Deviation C, P-7).
22. No new business rule ratified without an owner (authority class D).
23. Closed phase documents are **annotated, never rewritten** (`L-r-25`).
24. Phase 6 does not inherit `T5-nn` / `BLK-P5-nn` / `E-n` / `S3R-nnn` IDs (`01` §C.4 D-6).
25. Unproven restriction stores are flagged/excluded, never guessed; conflicts ⇒ `UNRESOLVED` (`L-r-17`, `L-r-18`).

---

## 4. Non-Goals / Out of Scope

### 4.1 Explicitly out of Phase 6 (`02` §B.3 boundary fence)

| Item | Authority |
|---|---|
| Schema authoring or migration creation | Deviation C, P-7, `L-r-23` |
| Deletion of the legacy `availability` table (G-1) | P-16 → **Phase 11** |
| Orphaned FO / guest-profile adapters (G-11) | Phase 5 §34 note 4 → **Phase 11** |
| Availability analytics / canonical occupancy contract (BD-6-09) | P-20, DS-06 → backlog |
| Outbound channel/CRS push publication (BD-6-10) | P-19, DS-07 → backlog |
| Admin / mobile availability surfaces | P-19 / M-7 greenfield; `01` K-18/K-19 = 0 files |
| ADR-072 reservation semantics | `02` §B.3 |
| Re-auditing or reopening Phase 5 | `01` P-1; `02` §0.4 |

### 4.2 Owned by other workstreams — must **not** enter this plan (`02A` §L.3 condition C4)

`02A` C4 names these verbatim as forbidden plan content: **OI-05, OI-06, OI-11, OI-27, OI-28, OI-29, OI-30.**

| ID | Item | Owning authority |
|---|---|---|
| OI-05 | `T5-36` GBA `pickupPct` client math vs `CLIENT_MATH_ALLOWED` (BD-6-07) | GBA / Reservations-frontend workstream |
| OI-06 | BLK-1 / BLK-2 wash deviations (BD-6-08) | GBA / wash workstream; Phase 4 carry-over |
| OI-11 | `T5-11` interim-gate honesty re-derivation (TD-6-08) | Phase 5 ownership |
| OI-27 | Duplicate `GET /tax-rates` merge (TD-6-06) | Hygiene, gated by OI-24 |
| OI-28 | Activation *rule* for `gba.reconciliation.enabled` / `twoLayerConsult` (TD-6-11) | Remains `DEFERRED` — see §13.4 |
| OI-29 | Allowlist / dead-DI removal *timing* (TD-6-13) | Removal rides with the retirement it accompanies |
| OI-30 | Analytics / channel publication | P-19, P-20 (ratified) |

### 4.3 Work verified as **not required** — do not re-do (`01` §P.1, `02` §O)

No rebuild of the assertion engine, snapshot, restriction evaluator, restriction write, or reconciliation service. No re-pointing of Reservations, Front Office, holds, or A3 restriction writes. No re-retirement of `engine/availability`, `engine/restrictions`, `engine/book`, `engine/release`, `rates/availability`. No re-do of `T5-50` (repository legacy leg) or `T5-43` (duplicate pages). No GitHub operations.

### 4.4 Out-of-plan observations

Findings encountered during targeting that **no decision covers** are recorded here rather than turned into tasks (`02` §R). None is currently blocking.

| Observation | Status |
|---|---|
| `availability-sales.controller.ts:728` carries a second `rtFilter` built against alias `rm`, distinct from the `:305` instance; it is consumed at `:735` inside a non-matrix query | **Not blocking.** No decision covers it. Record for the T6-05 implementer as an in-region observation; scope of T6-05 remains `TD-6-01` + `TD-6-10` as written. |

**If a genuinely blocking out-of-plan item is discovered during execution:** stop the affected task, record it as a planning blocker in `04_EXECUTION_EVIDENCE.md`, and escalate for a decision ID. Do not add it to this register.

---

## 5. Decision Traceability

Every task carries at least one authoritative decision or disposition. This table is the complete mapping; §7 restates it per task.

| Task | Title | Decision authority | Source finding / gate |
|---|---|---|---|
| T6-01 | Entry flag census & baseline record | `CO-6-06`, `CO-6-07`, `TD-6-12` | `01` J-AUD-01; `02A` §B.3 |
| T6-02 | E-1 six-store-family census (read-only) | `CO-6-01`, `TD-6-04` (step 1), `BD-6-02`, `TD-6-05` | `02A` OI-18; FDS §31 E-1 |
| T6-03 | E-3 runtime DI-chain confirmation | `CO-6-06`; FDS §28.3 ON-gate; BR-5-037 | `02A` OI-20; FDS §31 E-3 |
| T6-04 | E-5 external `/rates/engine/*` consumer inventory | `BD-6-03`, `TD-6-02`; BR-5-017/018/025 | `02A` OI-22; FDS §31 E-5; `phase-5/04:261,272` |
| T6-05 | Matrix `roomType` defect fix + raw-SQL parameterisation | **`TD-6-01` + `TD-6-10` (bundled — `02A` OI-08, OI-12 both `CLOSED`)** | `01` N-1, N-6; `02` §M M-r-01/02 |
| T6-06 | Typed reservation delete-reject | **`BD-6-06`** → `T5-59`; BR-5-043, INV-P5-22, AC-34 | `01` L-7, M-1; `02` §J |
| T6-07 | Phase 3 register annotation / re-verification | **`TD-6-07`** | `01` C-5.3, M-3 |
| T6-08 | FDS §24.1 stale-prose annotation | **`TD-6-12`** | `02` D-AUD-02 |
| T6-09 | Additive assertion-balance consumption field | **`TD-6-09`** via **02B Decision 3 (OI-07, Option A)** | `02` D-AUD-01, D-AUD-03 |
| T6-10 | Read-contract DB-backed verification + `/reconciliation` re-baseline | **`TD-6-09`** via **02B Decision 3**; `02A` §K.1 two-tier rule; `CO-6-04` (CLOSED) | `02` §H gate 3; `02A` §I capacity row |
| T6-11 | Findings-persistence mechanism + REQ-25.3 demonstration | **`TD-6-03`** (persistence half) | `02A` OI-15 `EXECUTION GATE ONLY`; `02` §G.3 row 5 |
| T6-12 | Scheduled report-only availability reconciler (projection) | **`TD-6-03`** (boundary + mechanism + cadence) | `01` I-1, I-7, N-4; `02` §G.1; `02A` OI-14 `CLOSED` |
| T6-13 | Counter-vs-balance comparator (I-2) | **`TD-6-03`**; `CO-6-02` lineage; B-1/B-2/B-3 | `01` I-2, X-3; `02` §G.1, §G.3 |
| T6-14 | Reservation population census (read-only) | **`TD-6-04`**, `CO-6-02`, `CO-6-03` | `01` I-3; `02` §H gate 5 |
| T6-15 | `LEGACY` assignment tooling (bounded batch) | **`TD-6-04`** + **D-21** (`L-r-22`) | `02` §G / §M M-r-08; `CO-6-02` |
| T6-16 | Retirement eligibility verification (RC-1…RC-5) | **`BD-6-03`** via **02B Decision 1 (OI-03)**; `CO-6-01` | FDS §24.3/§24.4; `02` §H gates 1,2,3 |
| T6-17 | Retire `POST /rates/engine/modify` + `crs.modifyReservation` | **`BD-6-03`** via **02B Decision 1**; **`TD-6-02`** | `01` F-5, G-3, H-1, N-2; `02` §H gate 6 |
| T6-18 | Remove `InventoryDomainService` + dead legacy methods | **`02` §I G-2** (REMOVE, gated by BD-6-03); B-1/B-3 | `01` F-6, F-7, F-8, G-2 |
| T6-19 | Remove dead repository DI + update scan-gate pin | **`02` §I G-16** (REMOVE after F-5 retires); `TD-6-13` | `01` G-16 |
| T6-20 | `gba.consumers.cascade` ON | **`BD-6-04`** (P-21) | `01` J-2, N-3, M-5; `02` §Q.2 item 7 |
| T6-21 | `gba.reconciliation.enabled` ON | **`BD-6-05`** via **02B Decision 2 (OI-04)**; `02` §H gate 8 | `02B` Decision 2 constraint 1; `02A` OI-28 stays `DEFERRED` |
| T6-22 | `gba.pickup.canonicalRead` ON + soak open | **`BD-6-05`** via **02B Decision 2**; `CO-6-05`; FDS §28.3/§28.5 | `01` J-6; `.env.example:105` |
| T6-23 | Soak window execution & monitoring | **`BD-6-05`** via **02B Decision 2** (SO-1…SO-4, F-1…F-5) | `02B` Decision 2 |
| T6-24 | Soak exit verification | **`BD-6-05`** via **02B Decision 2** | `02B` Decision 2 parameter 5 |
| T6-25 | `gba.pickup.canonicalWrite` ON + post-cutover verification | **`BD-6-05`** via **02B Decision 2**; `CO-6-05`; FDS §28.5 | `01` J-7; `02` §H gate 9 |
| T6-26 | Legacy `allotment_pickups` removal (G-10) | **`02` §I G-10** — `KEEP → REMOVE after canonicalWrite ON`; `BD-6-05` | `01` G-10; `02` §I |
| T6-27 | E-8 reconciled executed baselines | FDS §31 E-8; `02A` OI-25 `EXECUTION GATE ONLY` | `02` §Q.2 item 9 |
| T6-28 | §H cutover exit gates 1–9 evidence assembly | `02` §H gates 1–9; `CO-6-01…CO-6-07` | `02` §H.3 |

**Sequencing authorities cited by §8 but not owned by any task:** `DS-01`, `DS-03`, `DS-05` (FDS §24.2); `CO-6-05` (flag order); `02A` §K.1 (two-tier baseline rule).

---

## 6. Workstreams

Eight workstreams. They are **organisational groupings, not stages** — no gate, review, or ordering is implied by a workstream boundary, and none may be read as one (02B Decision 4: *"None introduced. No stage model"*).

| # | Workstream | Tasks | Purpose |
|---|---|---|---|
| **WS-1** | Entry Evidence & Preconditions | T6-01 … T6-04 | Produce the runtime evidence (`E-1`, `E-3`, `E-5`) and the as-is baseline that gates execution |
| **WS-2** | Authorised Technical Corrections | T6-05 … T6-08 | Corrections already authorised by the decision chain; no policy choice remains |
| **WS-3** | Canonical Read Contract | T6-09, T6-10 | OI-07 Option A implementation, DB-backed proof, re-baselining |
| **WS-4** | Reconciliation Closure | T6-11 … T6-13 | B-1's "dual-source reconciliation closure": scheduled report-only projection comparator + counter-vs-balance comparator |
| **WS-5** | Population Cutover | T6-14, T6-15 | B-1's "deterministic population migration at scale" under D-21 |
| **WS-6** | Legacy Writer Retirement | T6-16 … T6-19 | OI-03 gate-based EOL: eligibility → route retirement → `G-2` → `G-16` |
| **WS-7** | Controlled Canonical Cutover | T6-20 … T6-26 | `BD-6-04` + OI-04 soak sequence and `G-10` removal |
| **WS-8** | Exit Evidence | T6-27, T6-28 | E-8 baselines and the §H gates 1–9 completion record |

---

## 7. Task Register

Each task states the fourteen required fields. **Decision authority** is mandatory and non-empty for every task. **Verification** names a runnable check; **Evidence** names the `E6-nn` entry recorded in `04_EXECUTION_EVIDENCE.md` (§14).

Execution status vocabulary used below: `READY` (preconditions satisfiable now), `GATED` (scope fully defined; execution awaits a named precondition), `OPERATIONAL` (a configuration action, not a code change; performed outside any commit).

---

### WS-1 — Entry Evidence & Preconditions

#### T6-01 — Phase 6 entry flag census & baseline record

- **Objective:** Record the as-is flag and configuration census so that every later flag action has a documented starting state, and no task can mistake the documented end-state for the current state.
- **Decision authority:** `CO-6-06`, `CO-6-07`; `TD-6-12` (anchor re-verification duty); `01` J-AUD-01; `02A` §B.3.
- **Exact implementation scope:** Produce a census table covering all seven `gba.*` flags as found in `.env` / `.env.example:100-106` and in code defaults, plus the `RESERVATION_AVAILABILITY_PORT` and `RESTRICTION_EVALUATOR` bindings. Record the as-is row, not the documented row.
- **Files/modules/surfaces inspected:** `.env`, `.env.example:100-106`, `apps/api/src/modules/availability/availability.module.ts:44-56`, `apps/api/src/modules/group-allotment/**` flag reads, `apps/api/src/modules/shared/events.consumer.ts:150,171`.
- **Dependencies:** none. **Preconditions:** none — this is the entry record.
- **Required implementation behavior:** Census is **read-only**. No flag value is changed by this task (`CO-6-07`: flag flips are operational actions, never plan-authored side effects).
- **Verification:** census table matches direct re-read of `.env` and `.env.example`; every flag in `.env.example:100-106` appears exactly once; the single ON flag (`FEATURE_GBA_A3_AUTHORITATIVE=true`, `.env:68`) is identified as such.
- **Evidence:** `E6-01`.
- **Completion criteria:** census recorded; every subsequent flag task (T6-20 … T6-25) references it as its "current expected state".
- **Failure/stop condition:** if the live `.env` does not match either the as-is expectation above or `.env.example`, stop and record the discrepancy — do not proceed to any flag action until it is explained.
- **Rollback/recovery:** none needed (read-only).
- **Downstream tasks:** T6-20, T6-21, T6-22, T6-25, T6-28.

#### T6-02 — E-1 six-store-family census (read-only)

- **Objective:** Close the population half of `E-1` — per-hotel row counts, recency, and writer-or-UNKNOWN for each of the six store families — so `CO-6-01`'s precondition ("no legacy writer removal before E-1") can be satisfied and `BD-6-02` / `TD-6-05` have a denominator.
- **Decision authority:** `CO-6-01`; `TD-6-04` (census is its first step); `BD-6-02` and `TD-6-05` (both contingent on E-1); FDS §31 E-1; `phase-5/04:258` acceptance (*"store + count/recency/writer-or-UNKNOWN per hotel … no default assumptions"*).
- **Exact implementation scope:** A **read-only** census over: `room_inventory`, `rate_restrictions`, `out_of_order`, `out_of_service`, and the six A3 restriction tables (`close_to_arrival`, `close_to_departure`, `zero_sell_limits`, `room_type_sell_limits`, `minimum_length_of_stay`, `maximum_length_of_stay`). Output per store × hotel: row count, last-write recency, writer-or-UNKNOWN. The writer column is prefilled from `02A` OI-19 (`CLOSED`): **no in-repo writer** for `room_inventory`, `rate_restrictions`, `out_of_order`, `out_of_service`; the six A3 tables' writer is `PrismaRestrictionWriteRepository` via `RestrictionWriteService`.
- **Files/modules/surfaces inspected:** `packages/db/schema.prisma` (`:10413`, `:8925`, `:2056`, `:10087`); `apps/api/src/modules/availability/infrastructure/repositories/restriction-write.repository.ts:19-146`; `availability-source.adapter.ts:75-76`; `availability-sales.controller.ts:86,99,116,401,403`.
- **Dependencies:** `T6-01`. **Preconditions:** read-only DB access; **no schema change** (`L-r-23`).
- **Required implementation behavior:** `SELECT`-only. No default assumptions where a value is unknown — write `UNKNOWN`, never a guess (`phase-5/04:258`, `BR-5-046`).
- **Verification:** every store × hotel row is either a number + timestamp or an explicit `UNKNOWN`; the writer column for all four no-writer families matches `02A` OI-19; no `INSERT`/`UPDATE`/`DELETE` statement is present in the census tool.
- **Evidence:** `E6-02`.
- **Completion criteria:** census report recorded; `CO-6-01`'s E-1 precondition demonstrably satisfied for the store-family population.
- **Failure/stop condition:** if a store shows recent write activity from an unidentified source, record it as a live external-writer signal — this **gates** any legacy removal (`CO-6-01`) and escalates `BD-6-02` / `TD-6-05` steady state. **Do not proceed to T6-17 while an unidentified live writer is evidenced on any store the retirement touches.**
- **Rollback/recovery:** read-only; none.
- **Downstream tasks:** T6-14 (writer column carried forward), T6-16 (RC-1), T6-28 (§H gate 1).

#### T6-03 — E-3 runtime confirmation of the production DI chain

- **Objective:** Obtain executed evidence that the production dependency-injection chain is the one the specifications assume, closing the posture tension recorded at `02A` OI-20 (flag ON while FDS §28.3's four-condition gate is not fully evidenced).
- **Decision authority:** `CO-6-06`; FDS §28.3 ON-gate condition 3 (`03:1108`); BR-5-037; BR-5-009.
- **Exact implementation scope:** Recorded runtime observation, in a safe environment, of: `RESERVATION_AVAILABILITY_PORT → AvailabilityAssertionService` and `RESTRICTION_EVALUATOR → PrismaRestrictionAdapter`, together with request/response and rollback evidence.
- **Files/modules/surfaces inspected:** `apps/api/src/modules/availability/availability.module.ts:45,51` (bindings), `:50` (`UnresolvedRestrictionAdapter` retained rollback artifact — **must remain registered**).
- **Dependencies:** `T6-01`. **Preconditions:** safe environment with runtime access. **No code change** — this is observation only.
- **Required implementation behavior:** Observe; record; do not modify the bindings or the retained stub.
- **Verification:** observed binding matches `availability.module.ts:45` and `:51`; the stub at `:50` is confirmed still present (rollback line intact).
- **Evidence:** `E6-03`.
- **Completion criteria:** recorded request/response/rollback evidence filed.
- **Failure/stop condition:** if the observed binding differs from the module declaration, stop — this invalidates `DS-01` operational status and blocks T6-16 (RC-3).
- **Rollback/recovery:** observation only; no rollback needed. If the observation itself perturbs the environment, discard the environment and re-observe.
- **Downstream tasks:** T6-16 (RC-3), T6-28 (§H gate 1 via DS-01).

#### T6-04 — E-5 external `/rates/engine/*` consumer inventory

- **Objective:** Obtain the external-consumer inventory that `RC-2` requires, outside the repository, so `POST /rates/engine/modify` can become retirement-eligible.
- **Decision authority:** `BD-6-03` (02B Decision 1, RC-2); `TD-6-02`; BR-5-017/018/025; FDS §24.4; `phase-5/04:261,272` (T5-88 disposition: *"inventory unknown ⇒ in-repo retirements proceed, external-facing retirement waits (disposition unchanged)"*).
- **Exact implementation scope:** Query gateway / nginx / ops access logs or an ops consumer registry — **outside this repository** — for callers of `POST /api/v1/rates/engine/modify` and, for completeness, `GET /rates/engine/quote` and `GET /rates/engine/rate`. Record result, including an explicit negative result.
- **Files/modules/surfaces inspected (local, as corroboration only):** `gateway/ingress/nginx.conf`, `gateway/api-gateway/kong.yml` — both verified to contain **no consumer registry and no per-route consumer list** (`02A` OI-22). `apps/api/src/modules/rates-inventory/rates-inventory.controller.ts:34,46,108`.
- **Dependencies:** none (parallel with T6-02/T6-03). **Preconditions:** ops/gateway access outside the repo.
- **Required implementation behavior:** Evidence-gathering only. **Absence of a registry is not absence of traffic** — do not record "unused" from the repo result alone (`02A` OI-22).
- **Verification:** a dated inventory record exists naming, per endpoint, either specific external consumers or an explicit `NO EXTERNAL CONSUMER FOUND (source: <log/registry>, window: <range>)`.
- **Evidence:** `E6-04`.
- **Completion criteria:** inventory returned and recorded. If an external consumer exists, it must be **re-pointed first**; the route becomes eligible only when the **last** external consumer is gone (RC-2).
- **Failure/stop condition:** if the inventory cannot be obtained, **retirement is blocked indefinitely** — this is the intended conservative behaviour of the gate-based EOL (02B Decision 1, rationale 3). **Non-return blocks; it never permits.** Record as a planning/execution blocker with a decision ID; do not set a calendar date, and do not fall back to `02B` rejected Option B or C.
- **Rollback/recovery:** none (read-only).
- **Downstream tasks:** T6-16 (RC-2), T6-17, T6-28 (§H gate 2 via the E-5 branch).

---

### WS-2 — Authorised Technical Corrections

#### T6-05 — Matrix `roomType` defect fix + raw-SQL parameterisation (bundled)

- **Objective:** Eliminate the live defect where supplying `roomType` to `GET /availability/matrix` fails with *missing FROM-clause entry for table "rt"*, and remove the interpolation pattern that produced it, as **one** change.
- **Decision authority:** **`TD-6-01` (fix half — `RESOLVED BY TECHNICAL EVIDENCE`) + `TD-6-10` (parameterisation — `CLOSED` at `02A` OI-08/OI-12).** Bundling is mandatory: `02A` states *"plan carries one task, not two"*. Splitting is prohibited.
- **Exact implementation scope:**
  1. Correct the alias defect: `rtFilterRt` is built at `:306` as `AND UPPER(rt.room_type) = …` and interpolated at `:333` into a query whose only aliases are `rc`/`rd` (`:329-330`). The query executes unconditionally (`:326`), so it fails in **both** flag states.
  2. Correct the secondary misuse: `rtFilter` (`:305`, alias `r`) is valid for the query at `:320` but invalid at `:349` and `:400` — both `authoritative ? [] : …`, so flag-OFF only.
  3. Parameterise the interpolated filters/identifiers in this region: `rtFilter`/`rtFilterRt`/`rcFilter` (`:305-307`) and the direct `roomType` embeddings at `:377` and `:390`.
- **Files/modules/surfaces:** `apps/api/src/modules/activities/availability-sales.controller.ts` (`:290` route, `:305-307`, `:320`, `:326`, `:329-334`, `:349`, `:377`, `:390`, `:400`); existing pins `t38-authoritative-matrix.spec.ts`, `t520-matrix-projection-conformance.spec.ts`, `t528-fixed-surface-gate.spec.ts` (**these mock Prisma and catch neither defect** — a DB-backed test is required).
- **Dependencies:** none. **Preconditions:** DB-backed test environment.
- **Required implementation behavior:** `GET /availability/matrix?roomType=X` returns rows with no SQL error in **both** flag states; queries 3 and 6 correct under flag OFF; response shape unchanged (`t528` fixed-surface gate must remain green).
- **Verification:** one executed **DB-backed** regression test exercising the real query path with a `roomType` filter, run under `gba.a3.authoritative` ON and OFF; plus `pnpm --filter api typecheck`.
- **Evidence:** `E6-05`.
- **Completion criteria:** defect absent in both flag states under a real query path; parameterisation in place; `t528-fixed-surface-gate.spec.ts` still green.
- **Failure/stop condition:** any `t528` route-count change, or any regression in the mocked matrix specs → stop, revert the change, record as a blocker.
- **Rollback/recovery:** revert the single commit; no schema, flag, or data involved (`L-r-15`).
- **Downstream tasks:** T6-27 (E-8), T6-28 (§H gate 7).

#### T6-06 — Typed reservation delete-reject (`T5-59`)

- **Objective:** Make a ratified-but-unimplemented rule real: rejecting reservation delete with a deterministic typed business error **before any mutation** when availability state exists.
- **Decision authority:** **`BD-6-06`** → `RESOLVED BY EXISTING DOMAIN RULE` → `DIRECT TECHNICAL CORRECTION REQUIRED`. Ratified: BR-5-043, INV-P5-22 (`03:1149`), AC-34 (`03:1217`), FDS §26.4 (`:1046`), `L-r-20`. Source: `01` L-7, M-1; `02` §J.
- **Exact implementation scope:** Throw `RESERVATION_HAS_AVAILABILITY_STATE` (already defined at `apps/api/src/common/exceptions/availability-error-codes.ts:9,21` with HTTP `409`, currently **never thrown**) from `reservation.repository.ts` `delete()` (`:878`) **before any mutation**, when the reservation has availability-state rows. No availability operation runs in either branch.
- **Files/modules/surfaces:** `apps/api/src/modules/reservations/infrastructure/repositories/reservation.repository.ts:878-913`; `apps/api/src/common/exceptions/availability-error-codes.ts:9,21`; FK `Restrict` at `schema.prisma:17399,:17426` (unchanged); contract pin `t526-error-contract.spec.ts`.
- **Dependencies:** none. **Preconditions:** none.
- **Required implementation behavior:** (a) terminal reservation **with** availability-state rows → typed 409 before any mutation; balances and movement rows untouched; (b) reservation **without** such rows → delete performs no availability operation.
- **Verification:** unit test at the repository boundary covering the **AC-34 pair** (both branches), plus existing `t526-error-contract.spec.ts`.
- **Evidence:** `E6-06`.
- **Completion criteria:** `02` §Q.2 item 1 satisfied verbatim.
- **Failure/stop condition:** any path where the typed error is thrown *after* a mutation, or where balances/movements change → stop; this would violate `INV-P5-23` (append-only movements).
- **Rollback/recovery:** revert the single commit; no schema or data written by the check itself.
- **Downstream tasks:** Phase 9 Reservations notice (`02` §R — delete contract change); T6-27 (E-8).

#### T6-07 — Phase 3 register annotation / re-verification

- **Objective:** Prevent Phase 6 from consuming a stale upstream register, and record that it is stale, without rewriting a closed document.
- **Decision authority:** **`TD-6-07`** — `RESOLVED BY TECHNICAL EVIDENCE`: *"annotate, never rewrite; re-verify any Phase 3 register entry before consuming it."* `L-r-25`.
- **Exact implementation scope:** Attach a dated re-verification note — *"verified stale on \<date\> — see `docs/availability/phase-6/01_FORENSIC_AUDIT.md`"* — to the identified stale line(s), principally `docs/enterprise/availability-phase3-reservation-lifecycle-forensic-audit.md:756-761`, which states `holdInventory`/`confirmHold`/`releaseHold` are "no-op stubs … never injected" while `prisma-inventory-reservation.adapter.ts:37-70` implements holds via assertions and is pinned by `reservation-hold-mechanism.postgres.spec.ts`.
- **Files/modules/surfaces:** `docs/enterprise/availability-phase3-reservation-lifecycle-forensic-audit.md` (**annotation only**); cross-reference `01` C-5.3 / M-3.
- **Dependencies:** none. **Preconditions:** none.
- **Required implementation behavior:** Additive annotation. **The original text is not altered, deleted, or renumbered.**
- **Verification:** the note exists; a `git diff` of the file shows additions only, with no deletions of original lines.
- **Evidence:** `E6-07`.
- **Completion criteria:** every Phase 3 register entry consumed by Phase 6 carries a re-verification note or a fresh citation (`02` §M acceptance for TD-6-07).
- **Failure/stop condition:** if a rewrite (rather than an annotation) is required, stop — that is a `L-r-25` violation.
- **Rollback/recovery:** remove the added note.
- **Downstream tasks:** T6-08; T6-28.

#### T6-08 — FDS §24.1 stale-prose annotation (D-AUD-02)

- **Objective:** Correct the record that FDS §24.1 claims the reservation modify path writes **both** the assertion engine and the legacy counters "in one transaction … drift by design (F-04)", which post-`T5-50` is no longer true of the repository path.
- **Decision authority:** **`TD-6-12`** — `RESOLVED BY TECHNICAL EVIDENCE`; finding `D-AUD-02`. `L-r-25`.
- **Exact implementation scope:** Add a dated annotation to `docs/availability/phase-5/03_FINAL_DOMAIN_SPECIFICATION.md` §24.1 recording: the `:625` dual-write leg was removed by `T5-50`; `reservation.repository.ts` writes `replaceReservationAssertion` then `tx.reservations.update` with **no** legacy counter write; `this.crs` / `this.inventoryDomain` are never invoked; the remaining legacy writer is the **external** route only (M-8). **Annotate — do not rewrite.**
- **Files/modules/surfaces:** `docs/availability/phase-5/03_FINAL_DOMAIN_SPECIFICATION.md` §24.1 (**annotation only**); corroborated by `crs-engine.service.ts:391-402` and `reservation.repository.ts`.
- **Dependencies:** none. **Preconditions:** none.
- **Required implementation behavior:** Additive annotation only; original §24.1 prose untouched.
- **Verification:** `git diff` shows additions only; the annotation quotes M-8 rather than re-deriving it.
- **Evidence:** `E6-08`.
- **Completion criteria:** any Phase 6 document quoting `:625` now quotes M-8 instead (`02` §E TD-6-12 downstream).
- **Failure/stop condition:** a rewrite of §24.1 → stop (`L-r-25`).
- **Rollback/recovery:** remove the added note.
- **Downstream tasks:** T6-28.

> **Standing duty — anchor re-verification.** `TD-6-12` makes re-verification of `path:line` anchors a **standing execution duty**, not a one-off (`02` §U row 5). Every task in this plan re-verifies its own anchors at execution time and records any drift as evidence, rather than assuming the anchors in this document are still current. Anchors here were verified against the local tree while authoring (§2.3) and must be re-verified before each use.

---

### WS-3 — Canonical Read Contract (OI-07)

#### T6-09 — Additive assertion-balance consumption field

- **Objective:** Make the canonical Availability read materialise `availability_assertion_balances` for `ASSERTION_MANAGED` reservations, so `snapshot.sellableAvailable`, the matrix, and `/reconciliation`'s `expected` agree with the already-correct write gate.
- **Decision authority:** **`TD-6-09` as finalized by `02B` Decision 3 (OI-07) — Option A.** Engineering owner sign-off with product acknowledgement (operator-visible number). Ratified support: P-11, P-12, FDS §26.2, G5.
- **Exact implementation scope (exactly as constrained by `02B`, no more):**
  1. Add a **provenance-labelled additive field** — `assertionBalanceConsumption` — read from `availability_assertion_balances`.
  2. Combine it in `apps/api/src/modules/availability/domain/policies/snapshot-calculator.ts` into `consumption` / `sellableAvailable`.
  3. Leave the existing `reservationConsumption` field **byte-identical** — no existing consumer may break.
  4. **G5 preserved:** the legacy source still excludes `ASSERTION_MANAGED` rows; the balance term comes from the balances store, **never** from re-counting reservations.
  5. Add a dated **amendment/annotation note against FDS §11.3** (`docs/availability/phase-5/03_FINAL_DOMAIN_SPECIFICATION.md`) — annotate, never rewrite (`TD-6-07`, `L-r-25`).
- **Files/modules/surfaces:** `availability/domain/policies/snapshot-calculator.ts:14`; `application/services/availability-snapshot.service.ts:88,98`; `infrastructure/adapters/availability-source.adapter.ts:140`; `infrastructure/adapters/reservation-consumption.adapter.ts:20-22` (**unchanged — G5**); a new read of `availability_assertion_balances`; `availability-sales.controller.ts:494,539` (matrix `available` moves in lockstep); `phase-5/03_FINAL_DOMAIN_SPECIFICATION.md` §11.3 (**annotation only**).
- **Dependencies:** none technically. **Preconditions:** engineering owner sign-off + product acknowledgement, per `02A` OI-07 authority split.
- **Required implementation behavior:** For a property with N `ASSERTION_MANAGED` reservations and no legacy population, `snapshot.sellableAvailable = capacity − N` (today it equals `capacity`). Matrix and snapshot agree under both flag states. No existing field's shape changes.
- **Explicit prohibition carried from `02B`:** **individual readers must not be required to subtract balances themselves.** That option is `RULED OUT` by P-11/P-12 and is not reopened. The combination happens **once**, in the snapshot calculator.
- **Verification:** implementation-level checks plus `pnpm --filter api typecheck`; the definitive proof is T6-10.
- **Evidence:** `E6-09`.
- **Completion criteria:** additive field present; calculator combines it; `reservationConsumption` byte-identical; FDS §11.3 annotation dated and recorded; **no flag change implied** (`G-5` stands).
- **Failure/stop condition:** if any existing consumer's payload shape changes, or if `reservationConsumption` semantics change rather than being left byte-identical → stop and revert. If G5 would be violated (any double count) → stop.
- **Rollback/recovery:** revert the commit; no schema, migration, flag, or data write is involved. `02A` OI-07's conditional fallback branch (documented-gap note + withheld capacity baselines) remains the recorded alternative if the change cannot be completed — **reverting here means capacity-grade baselines stay withheld**, per `02A` §K.1.
- **Downstream tasks:** T6-10, T6-12, T6-27, T6-28 (§H gate 3); Phase 9 Reservations notification (`02` §R).

#### T6-10 — Read-contract DB-backed verification + `/reconciliation` re-baseline

- **Objective:** Prove the OI-07 change with DB-backed evidence, and invalidate/re-capture every `/reconciliation` baseline so no stale baseline is mistaken for evidence.
- **Decision authority:** **`TD-6-09` / 02B Decision 3**; `02A` §K.1 two-tier baseline rule; `CO-6-04` (CLOSED by `02A`); `02` §R testing policy.
- **Exact implementation scope:** Three **DB-backed** tests, one per surface, each seeding `ASSERTION_MANAGED` balances and proving the delta: **snapshot**, **matrix**, **`/reconciliation`**. Then re-baseline `/reconciliation` and classify each baseline under `02A` §K.1's two tiers:
  - **Projection-drift** baselines (the comparator's ratified purpose, REQ-25.1) — re-captured after the change.
  - **Absolute sellable-capacity** baselines — now obtainable **because** OI-07 is implemented.
  - **Assertion-integrity** evidence — still requires **I-2** (T6-13) and is **not** substituted by this task.
- **Files/modules/surfaces:** new DB-backed specs alongside existing `*.postgres.spec.ts` convention; `availability-reconciliation.service.ts:22-24`; existing mocked specs (`t38`, `t520`, `t510-parity-soak.spec.ts`) are explicitly **insufficient** and are not used as proof.
- **Dependencies:** `T6-09`. **Preconditions:** DB-backed test environment; T6-09 complete.
- **Required implementation behavior:** Seed balances; assert `capacity − N`; assert matrix↔snapshot parity preserved (both move together); assert `/reconciliation` `expected` equals the same figure; assert no existing field shape changed.
- **Verification:** three executed DB-backed tests, green.
- **Evidence:** `E6-10` (includes the re-baselined baseline set).
- **Completion criteria:** `02` §Q.2 item 3 satisfied; `02` §H gate 3 ("TD-6-09 confirmed and, if approved, implemented with DB-backed proof") **satisfiable**; `02A` §K.1 capacity-tier gate released.
- **Failure/stop condition:** any of the three surfaces disagreeing, or matrix↔snapshot parity breaking → stop; treat as a failed OI-07 implementation and revert to T6-09 rollback. **Capacity-grade baselines remain withheld while this task is not complete.**
- **Rollback/recovery:** revert T6-09 and T6-10 together; return to the `02A` §K.1 state where projection baselines may be captured and capacity baselines may not.
- **Downstream tasks:** T6-12, T6-13, T6-27, T6-28 (§H gate 3).

---

### WS-4 — Reconciliation Closure

#### T6-11 — Findings-persistence mechanism selection + REQ-25.3 boundary demonstration

- **Objective:** Choose where reconciliation findings persist under the **no-schema-change** constraint, and demonstrate that the chosen mechanism cannot feed a decision or write domain state.
- **Decision authority:** **`TD-6-03`** (persistence half); `02A` **OI-15 = `EXECUTION GATE ONLY`**; `02` §G.3 row 5. Ratified boundary: REQ-25.3, FDS §26.2 layer 3, Deviation C / `L-r-23`.
- **Exact implementation scope:** Select an existing non-domain persistence mechanism (no new table — `L-r-23` forbids schema authoring) that satisfies: hotel-scoped, outside domain state, held in the **FDS §26.2 layer 3 reconciliation-evidence** layer. Demonstrate by test/inspection that no availability decision, eligibility computation, or publication path reads it, and that no write statement is reachable from it.
- **Files/modules/surfaces:** `apps/api/src/modules/availability/infrastructure/reconciliation/`; `docs/availability/phase-5/03_FINAL_DOMAIN_SPECIFICATION.md:1030-1037` (four-layer table), `:1020` (REQ-25.3), `:1024-1026` (§26.1 canonical log routes); pins `t557-reconciliation-readonly.spec.ts`, `t561-logs-never-authority.spec.ts`.
- **Dependencies:** none. **Preconditions:** Deviation C acknowledged (no schema change).
- **Required implementation behavior:** Selection is an **execution-time technical choice** (`02A` OI-15) — it is not an owner decision and not an evidence gap. It must be recorded before implementation begins.
- **Verification:** an explicit REQ-25.3 acceptance test: findings are readable only from a reconciliation-evidence path; no consumer feeds an availability decision; the write-path pin (`t557` expects exactly `['availability.findMany']`) still holds or its extension is proven read-only.
- **Evidence:** `E6-11`.
- **Completion criteria:** mechanism named and recorded; REQ-25.3 boundary demonstrated.
- **Failure/stop condition:** if the only viable mechanism would require a schema change, **stop** — `L-r-23` is non-negotiable; escalate with a decision ID rather than authoring a table.
- **Rollback/recovery:** if the mechanism proves unsound, revert T6-12/T6-13 implementation; the boundary rule itself is unchanged.
- **Downstream tasks:** T6-12, T6-13.

#### T6-12 — Scheduled report-only availability reconciler (projection comparator)

- **Objective:** Give the existing sanctioned comparator (`I-1`) a schedule and a findings store, so projection drift is continuously observed rather than visible only on demand.
- **Decision authority:** **`TD-6-03`** — boundary `RESOLVED BY EXISTING DOMAIN RULE` (REQ-25.x), mechanism `RESOLVED BY TECHNICAL EVIDENCE` (scheduled report-only job), cadence `CLOSED` at `02A` OI-14 (**hourly, matching the GBA precedent, behind a flag default OFF per P-21**), event-vs-poll `CLOSED` at `02A` OI-09 (**scheduled poll; outbox event not required**).
- **Exact implementation scope:** Extend `AvailabilityReconciliationService` with a scheduled run following the `gba-reconciliation.service.ts:131` precedent (`@Cron(CronExpression.EVERY_HOUR)`), gated by a configuration flag defaulting **OFF** (P-21; exact key chosen at execution following the `FEATURE_*` convention at `.env.example:100-106`). Persist findings via the mechanism chosen in T6-11. `automaticRepair` remains literal `false`.
- **Files/modules/surfaces:** `apps/api/src/modules/availability/infrastructure/reconciliation/availability-reconciliation.service.ts:18,22,23,24,32`; `apps/api/src/modules/availability/api/controllers/availability.controller.ts:21-28` (route unchanged); no `@Cron` currently exists under `modules/availability` (`01` I-7). Precedent: `gba-reconciliation.service.ts:131,162-201`.
- **Dependencies:** `T6-11` (mechanism), `T6-10` (so `expected` is final — the read contract moves the figure).
- **Preconditions:** T6-11 complete; T6-10 complete so baselines are not captured against a moving `expected`.
- **Required implementation behavior:**
  - Cadence: hourly, parameterised so it can be tuned to `BD-6-05`'s soak interval (`02A` OI-14).
  - Comparison: legacy `availability.available` (observed, `authoritative: false`) vs canonical `sellableAvailable` (expected).
  - Classification (existing, unchanged): `MATCH` · `DIAGNOSTIC_VARIANCE` · `MISSING_PROJECTION` · `UNRESOLVED_SOURCE` (`availability-reconciliation.service.ts:24`).
  - `automaticRepair: false` in every configuration, every run, forever.
  - Flag default OFF; **flipped as an operational action with gate evidence, never inside a commit** (`L-r-16`).
- **Verification:** one executed scheduled run producing a persisted report; a proof that **no write statement executed** (extend the `t557` read-only pin); `t561` "no consumers besides its own route" still green; `pnpm --filter api typecheck`.
- **Evidence:** `E6-12`.
- **Completion criteria:** `02` §Q.2 item 4 satisfied verbatim — *"a scheduled run exists; findings persist as reconciliation evidence; no write statement executes; output never reaches an availability decision."*
- **Failure/stop condition:** any write reachable from the reconciler, or any path where findings feed an availability decision → stop immediately (REQ-25.3 breach).
- **Rollback/recovery:** set the reconciler flag OFF (default state) — the job stops; no data is modified because nothing was written. Revert the commit if the code itself is unsound.
- **Downstream tasks:** T6-13, T6-27 (E-8), T6-28 (§H gate 4).

#### T6-13 — Counter-vs-balance comparator (I-2)

- **Objective:** Build the **absent** counter-vs-balance comparator — the literal "dual-source reconciliation closure" of B-1/B-2/B-3 — as report-only evidence.
- **Decision authority:** **`TD-6-03`**; `02` §G.1 (`I-2` → **BUILD**, report-only); `02` §G.3 (*"Is counter-vs-balance comparison in scope? Yes — it is B-1/B-2/B-3 verbatim"*); `02` X-3 part 2; `02A` §I row *"Counter-vs-balance comparator build"*.
- **Exact implementation scope:** An additional **sanctioned, read-only** comparison, introduced with the same boundary as `REQ-25.1` — never as a second truth (`02` §G.2 inv. 4).

  **Population:** every `(hotel_id, room_type, stay_date)` in the union of the legacy `availability` rows and the `availability_assertion_balances` rows within the requested scope. Scope is hotel-scoped and date-bounded, matching the existing route.

  **Comparison source — legacy side:** `availability.reserved` (`schema.prisma:439`) — the legacy counter's representation of committed rooms. Stored with `authoritative: false`.

  **Comparison source — authority side:** `availability_assertion_balances.asserted_quantity` (`schema.prisma:17320`).

  **Expected/result interpretation:** equality means the two representations of reservation commitment agree for that key. Inequality is a **flagged fact** describing the magnitude of the `H-2` dual-source divergence — it is evidence of drift, not a defect to be corrected by the comparator.

  **Mismatch classification:**

  | Class | Condition |
  |---|---|
  | `MATCH` | `availability.reserved == asserted_quantity` |
  | `COUNTER_VARIANCE` | both sides present and unequal |
  | `MISSING_LEGACY_COUNTER` | no `availability` row for the key but balance `> 0` |
  | `MISSING_BALANCE` | balance absent/zero but `availability.reserved > 0` |
  | `UNCOMPARABLE` | neither side present — no row emitted, recorded as scope-noise only |

  **Remediation path — none authorised by this comparator.** `REQ-25.2.1/.4` forbid auto-repair; `REQ-25.3` forbids the output feeding a decision. The only remediation paths in Phase 6 are the separately gated ones: retire the `H-1` generator (T6-17), first-touch population conversion (T6-15), and human investigation recorded as evidence. **Reconciliation output may not trigger any of them automatically.**

- **Files/modules/surfaces:** `packages/db/schema.prisma:433-453` (`availability`), `:17316-17330` (`availability_assertion_balances`); comparator placed alongside `availability-reconciliation.service.ts`; pins `t557-reconciliation-readonly.spec.ts`, `t561-logs-never-authority.spec.ts`.
- **Dependencies:** `T6-11` (mechanism); `T6-10` (re-baselined expectations elsewhere; independent read path).
- **Preconditions:** T6-11 complete; no schema change (`L-r-23`).
- **Required implementation behavior:** `SELECT`-only; `automaticRepair: false`; hotel-scoped; findings persisted in the T6-11 mechanism; output never read by any availability decision path.
- **Verification:** one executed run producing persisted findings; proof no write statement executed; classification table covered by tests for all five classes.
- **Evidence:** `E6-13`.
- **Completion criteria:** `02` §H gate 4 satisfied — *"counter-vs-balance comparator running report-only with persisted findings."*
- **Failure/stop condition:** any write, any repair attempt, or any consumer feeding an availability decision → stop (REQ-25.2 / REQ-25.3).
- **Rollback/recovery:** flag OFF (default) stops scheduled runs; revert the commit if unsound. No data was written.
- **Downstream tasks:** T6-27, T6-28 (§H gate 4); supplies the assertion-integrity tier of `02A` §K.1 that this task alone can produce.

> **§9.4 records the documented register-wording discrepancy for this comparison.**

---

### WS-5 — Population Cutover

#### T6-14 — Reservation population census (read-only)

- **Objective:** Establish the population denominator — how many reservations are `LEGACY`, `ASSERTION_MANAGED`, or have no state row — before any assignment tooling exists.
- **Decision authority:** **`TD-6-04`** (`RESOLVED BY EXISTING DOMAIN RULE` D-21 + tooling `RESOLVED BY TECHNICAL EVIDENCE`); `CO-6-02` (*"Read-only census (denominator), then `LEGACY` assignment tooling, then first-touch conversion"*); `CO-6-03`; `02` §H gate 5.
- **Exact implementation scope:** A **read-only** census over `reservation_availability_state` (`schema.prisma:17391`) reporting counts by `population` (`LEGACY` / `ASSERTION_MANAGED`) and a separate **no-row** count against the reservations population, hotel-scoped.
- **Files/modules/surfaces:** `packages/db/schema.prisma:17391-17404`; `apps/api/src/modules/availability/application/services/availability-assertion.service.ts:563-570` (`ensureAndLockState` first-touch), `:552` (population assignment), `:272-273` (fail-closed gate).
- **Dependencies:** `T6-01`; `T6-02` (writer column carried forward, per `02A` OI-18).
- **Preconditions:** read-only DB access; **no schema change**.
- **Required implementation behavior:** `SELECT`-only. **No default assumptions** (`phase-5/04:258`).
- **Verification:** report exists with a count for each of the three buckets per hotel; totals reconcile against the reservations count; no write statement present.
- **Evidence:** `E6-14`.
- **Completion criteria:** `02` §H gate 5 satisfied — *"population census produced; `LEGACY` / `ASSERTION_MANAGED` / no-row counts known."*
- **Failure/stop condition:** if counts cannot be produced (e.g. a hotel's state rows are unreadable), stop — T6-15 may not start without a denominator (CO-6-02).
- **Rollback/recovery:** read-only; none.
- **Downstream tasks:** T6-15, T6-28 (§H gate 5).

#### T6-15 — `LEGACY` assignment tooling (bounded batch)

- **Objective:** Provide deterministic population migration **at scale** without bulk backfill, so `RESERVATION_POPULATION_MISMATCH` becomes meaningful and the split becomes explicit.
- **Decision authority:** **`TD-6-04`** + **D-21** (`L-r-22`: first-touch assign, sticky, **no bulk backfill**; `LEGACY` refuses assertion fail-closed); `CO-6-02`; `CO-6-03`.
- **Exact implementation scope:** A **new, reviewed, transactional** tool that writes **only** `reservation_availability_state`, assigning `LEGACY` to in-flight reservations not yet touched, in bounded batches. It writes **no balance rows and no movement rows**, ever.
- **Files/modules/surfaces:** `packages/db/schema.prisma:17391` (target table — **no schema change**); `availability-assertion.service.ts:271-275` (first-touch converts on next assertion), `:552`, `:632` `setPopulationInTransaction` on the port `reservation-availability.port.ts:71`; guard behaviour at `:273` → `RESERVATION_POPULATION_MISMATCH` 409.
- **Dependencies:** `T6-14` (denominator — mandatory, CO-6-02). **Preconditions:** census recorded; no schema change; `L-r-22` respected.
- **Required implementation behavior:**
  - Idempotent — re-running a batch produces no further change.
  - Hotel-scoped on every predicate.
  - Transactional.
  - Writes **only** `reservation_availability_state`.
  - **Never** creates balances or movements.
  - Reversed only by re-assertion (first-touch), never by a reverse-write path.
  - **Bulk backfill is prohibited** (D-21) — bounded batches only.
  - `RESERVATION_POPULATION_MISMATCH` stays fail-closed.
- **Verification:** (a) dry-run diff showing exactly which rows would change; (b) idempotency proof — run twice, second diff empty; (c) proof that a `LEGACY` reservation is refused until first-touch; (d) proof no balance/movement row was written.
- **Evidence:** `E6-15`.
- **Completion criteria:** `02` §Q.2 item 5 satisfied verbatim — *"the assignment tool writes only `reservation_availability_state`, is idempotent and hotel-scoped, and never creates balances or movements; a `LEGACY` reservation is refused with `RESERVATION_POPULATION_MISMATCH` until first-touch."*
- **Failure/stop condition:** any write outside `reservation_availability_state`; any bulk backfill; any balance/movement creation → stop immediately (D-21, `INV-P5-23`).
- **Rollback/recovery:** re-assertion restores `ASSERTION_MANAGED` on next touch (the only sanctioned reversal); the tool itself is reverted by commit rollback.
- **Downstream tasks:** T6-28 (§H gate 5); feeds the I-2 population definition (T6-13).

---

### WS-6 — Legacy Writer Retirement

#### T6-16 — Retirement eligibility verification (RC-1 … RC-5)

- **Objective:** Prove, before any removal, that `POST /rates/engine/modify` satisfies all five gate-based EOL conditions fixed by `02B` Decision 1.
- **Decision authority:** **`BD-6-03` as finalized by `02B` Decision 1 (OI-03) — Option A, gate-based EOL, no calendar date.** `CO-6-01`; FDS §24.3 (BR-5-026), §24.4 (BR-5-018/025).
- **Exact implementation scope:** Verify and record each condition:

  | # | Condition | Evidence source | Verifying task |
  |---|---|---|---|
  | **RC-1** | E-1 census complete — per-hotel counts, recency, writer identification for all six store families | `E6-02` | T6-02 |
  | **RC-2** | E-5 answered — external consumer inventory returned; last external consumer gone (re-point first if any) | `E6-04` | T6-04 |
  | **RC-3** | DS-01 operational — evaluator bound + E-1 + E-3 + parity soak recorded | `E6-02`, `E6-03`; bindings at `availability.module.ts:45,51`; Phase 5 parity soak record | T6-02, T6-03 |
  | **RC-4** | `TD-6-02` disposition executed — no path mutates `reservations` stay fields without the assertion port | see §8.4 Interpretation 1 / §17 row 1 | T6-16 (pre), T6-17 (post) |
  | **RC-5** | In-repo consumers = ∅ | `rg -n "engine/modify" apps/api/src apps/web` → only the route definition (re-run) | T6-16 |

- **Files/modules/surfaces:** `rates-inventory.controller.ts:108`; `crs-engine.service.ts:290`; `reservation.repository.ts` (repository path must be confirmed port-only); `check-out.handler.ts`, `upgrade-room.handler.ts`, `prisma-inventory-reservation.adapter.ts:37-70` (the paths that must already conform).
- **Dependencies:** `T6-02`, `T6-03`, `T6-04`. **Preconditions:** all of `E6-02`, `E6-03`, `E6-04` recorded.
- **Required implementation behavior:** Verification only — **no removal happens in this task.**
- **Verification:** a recorded eligibility statement listing RC-1…RC-5 with `SATISFIED` / `NOT SATISFIED` and the evidence reference for each. Any `NOT SATISFIED` blocks T6-17.
- **Evidence:** `E6-16`.
- **Completion criteria:** all five recorded `SATISFIED`. This is the entry gate for T6-17.
- **Failure/stop condition:** any condition not satisfied → **T6-17 may not start.** Per `02B`: *"If a condition cannot be met, that is a new escalation requiring its own decision ID — nothing enters Phase 6 without one."* **No fallback is pre-authorised, and no calendar date may be set** (02B rejected Options B and C).
- **Rollback/recovery:** none (read-only).
- **Downstream tasks:** T6-17, T6-28 (§H gates 1, 2, 3).

#### T6-17 — Retire `POST /rates/engine/modify` + `crs.modifyReservation`

- **Objective:** Eliminate the `H-1` dual-truth generator and execute `TD-6-02`'s disposition.
- **Decision authority:** **`BD-6-03` via `02B` Decision 1**; **`TD-6-02`** — disposition `RESOLVED BY EXISTING DOMAIN RULE` (BR-5-017/018/025, §21.4, §24.4) = **retire**; `02` §H gate 6. `T5-88` disposition governs the E-5 dependency.
- **Exact implementation scope:**
  1. Remove `POST engine/modify` (`rates-inventory.controller.ts:108`) so it returns **404**.
  2. Remove `CrsEngineService.modifyReservation` (`crs-engine.service.ts:290-444`), including the legacy counter leg `:391-402`, the ungated `tx.reservations.update` `:411`, and the `reservation_changes` insert `:425`.
  3. Remove the `G-15` retired-route delay allowlist entries that ride with this route: `apps/web/app/(dashboard)/rates-inventory/page.tsx:140`, `apps/web/features/reservations/hooks/use-crs-book.ts:91`, `apps/web/features/front-office/api/front-office.api.ts:193` (`TD-6-13`: each removal rides with the retirement it accompanies).
  4. Re-run the in-repo consumer scan and record the result.
- **Files/modules/surfaces:** `apps/api/src/modules/rates-inventory/rates-inventory.controller.ts:108`; `apps/api/src/modules/rates-inventory/crs-engine.service.ts:290-444`; `apps/web/app/(dashboard)/rates-inventory/page.tsx:140`; `apps/web/features/reservations/hooks/use-crs-book.ts:91`; `apps/web/features/front-office/api/front-office.api.ts:193`; retirement pins `t523`/`t524`/`t525` **must be updated** — they currently assert `@Post('engine/modify')` *exists*.
- **Dependencies:** `T6-16` (all five RC satisfied). **Preconditions:** `E6-16` shows RC-1…RC-5 `SATISFIED`; no unidentified live legacy writer from `E6-02`.
- **Required implementation behavior:** The legacy writer must **not** remain an accepted final state (FDS §24.4). Retire it; do not deprecate, do not harden-then-leave (`02B` rejected Options B and C). `H-1` must have **no remaining generator**.
- **Verification:** (a) `POST /api/v1/rates/engine/modify` returns 404; (b) `rg -n "engine/modify" apps/api/src apps/web` returns **zero** non-test occurrences (test pins updated to assert absence); (c) `rg -n 'assertion' apps/api/src/modules/rates-inventory/crs-engine.service.ts` — if the service still exists, it must no longer expose a path that mutates `reservations` stay fields outside the port; (d) `pnpm --filter api typecheck`; (e) `t522`–`t525` retirement specs green in their updated form.
- **Evidence:** `E6-17`.
- **Completion criteria:** `02` §Q.2 item 6 satisfied — *"no in-repo path mutates `reservations` stay fields without `RESERVATION_AVAILABILITY_PORT`; `rg engine/modify` returns … zero callers."* `02` §H gate 6 satisfied (H-1 generator eliminated); RC-4 now verifiable in full (§8.4 Interpretation 1).
- **Failure/stop condition:** any residual caller; any remaining path mutating `reservations` stay fields without the port; any regression in `t528`/`t522`-`t525` → stop, revert.
- **Rollback/recovery:** revert the commit — the route and service return. **No schema, flag, or data change accompanies this task**, so rollback is code-only (`L-r-15`). Note that rollback **restores the `H-1` generator**: if rolled back, `02B` Decision 1 eligibility must be re-recorded before any further attempt.
- **Downstream tasks:** T6-18, T6-19, T6-28 (§H gate 6); Reservations Phase 9 notice (`H-7` divergence disappears — `02` §R).

#### T6-18 — Remove `InventoryDomainService` + dead legacy methods

- **Objective:** Remove the legacy counter writer machinery once its sole live caller is gone.
- **Decision authority:** **`02` §I `G-2` = `REMOVE`, gated by `BD-6-03`** (its only live caller is `engine/modify`); B-1/B-3 *"removal of `crs-engine` inventory calls"*.
- **Exact implementation scope:** Remove `InventoryDomainService` (`inventory.domain-service.ts:56-286`) including: `checkAvailability` (`:56`), `reserve` (`:160`), `release` (`:177`), and the dead zero-caller methods `reserveRooms` (`:190`), `releaseRooms` (`:198`), `blockAvailability` (`:209`), `consumePickup` (`:238`), `releaseUnsold` (`:267`) — all raw `UPDATE availability` sites (`:182, :228, :257, :276`). Also remove the now-dead `crs-engine.checkAvailability` (`crs-engine.service.ts:86-93`) if it has no remaining caller.
- **Files/modules/surfaces:** `apps/api/src/modules/rates-inventory/domain/services/inventory.domain-service.ts`; `apps/api/src/modules/rates-inventory/crs-engine.service.ts:86-93`; `reservation.repository.ts:5,15,83,89` (injection removal is **T6-19**, not here); scan-gate allowlist in `t525-engine-release-retirement.spec.ts` must be updated to reflect removal.
- **Dependencies:** `T6-17`. **Preconditions:** T6-17 complete; caller scan shows zero live callers.
- **Required implementation behavior:** Removal only after the caller is gone. **The legacy `availability` table (G-1) is NOT touched** — it remains `KEEP` read-only to Phase 11 (P-16).
- **Verification:** `rg -n 'InventoryDomainService' apps/api/src` → only test/scanner references or zero; `rg -n 'UPDATE availability' apps/api/src` → zero in production source; `pnpm --filter api typecheck`.
- **Evidence:** `E6-18`.
- **Completion criteria:** zero production callers; zero raw `UPDATE availability` statements outside tests; `G-2` disposition executed.
- **Failure/stop condition:** any remaining live caller → stop (this would violate `CO-6-01` sequencing). `t557-reconciliation-readonly.spec.ts`'s `AVAILABILITY_TABLES` scanner expectations must be re-checked and updated only to reflect removal, never relaxed.
- **Rollback/recovery:** revert the commit. No schema or data change.
- **Downstream tasks:** T6-19, T6-28.

#### T6-19 — Remove dead repository DI + update scan-gate pin

- **Objective:** Remove the dead `this.crs` / `inventoryDomain` dependency injection in the reservation repository now that F-5 has retired, and update the pin that currently records it as intentional.
- **Decision authority:** **`02` §I `G-16` = `REMOVE` after F-5 retires**; `TD-6-13` (removal rides with the retirement).
- **Exact implementation scope:** Remove `InventoryDomainService` (and any now-dead `crs`) injection from `reservation.repository.ts` (`:5`, `:15`, `:83`, `:89`), and update the `ws-n-scan-gates.spec.ts` pin (`:187`) that currently records the dead DI as intentional.
- **Files/modules/surfaces:** `apps/api/src/modules/reservations/infrastructure/repositories/reservation.repository.ts`; `apps/api/src/modules/availability/infrastructure/__tests__/ws-n-scan-gates.spec.ts`.
- **Dependencies:** `T6-17`, `T6-18`. **Preconditions:** both complete.
- **Required implementation behavior:** Hygiene only — **no behaviour change**. The repository must remain authority-only (`replaceReservationAssertion` + `tx.reservations.update`, no counter write).
- **Verification:** `rg -n 'inventoryDomain' apps/api/src/modules/reservations` → zero; repository still writes no legacy counter; `ws-n-scan-gates.spec.ts` green with the updated expectation; `pnpm --filter api typecheck`.
- **Evidence:** `E6-19`.
- **Completion criteria:** dead DI removed; pin updated to record removal rather than intentionality.
- **Failure/stop condition:** any behaviour change in the repository delete/write paths → stop (would endanger T6-06 and the `L-r-21` port-only rule).
- **Rollback/recovery:** revert the commit.
- **Downstream tasks:** T6-28.

---

### WS-7 — Controlled Canonical Cutover

> **Invariant:** `canonicalRead` → soak → `canonicalWrite`. Never altered. Never both ON (`CO-6-05`, FDS §28.3/§28.5, P-21). Every flag action below is `OPERATIONAL` — performed outside any commit, with recorded gate evidence (`L-r-16`).

#### T6-20 — `gba.consumers.cascade` ON (deploy invariant)

- **Objective:** Restore the ratified deploy posture so the pickup cascade actually writes the canonical ledger — a precondition of any canonical read having data to read.
- **Decision authority:** **`BD-6-04`** — `RESOLVED BY EXISTING DOMAIN RULE` (P-21 already decides it; only execution is outstanding).
- **Exact implementation scope:** Set `FEATURE_GBA_CONSUMERS_CASCADE=true` in deploy configuration, with a recorded gate note.
- **Files/modules/surfaces:** deploy `.env` (currently **absent → false**, `.env:68` area); `.env.example:101` (*"DEPLOY INVARIANT: MUST be ON at deploy"*); consumers `apps/api/src/modules/shared/events.consumer.ts:150,171`.
- **Dependencies:** `T6-01` (baseline recorded). **Preconditions:** release window; `T6-01` complete.
- **Required implementation behavior:** Configuration change only, **no code**. **Never inside a code commit** (`L-r-16`, G-5).
- **Verification:** post-flip evidence record; observed pickup cascade firing (`events.consumer.ts:151,172` no longer taking the `debug` skip branch).
- **Evidence:** `E6-20`.
- **Completion criteria:** `02` §Q.2 item 7 satisfied — *"`FEATURE_GBA_CONSUMERS_CASCADE=true` recorded with a gate note, changed outside any commit."*
- **Failure/stop condition:** flag changed inside a commit, or cascade observed not to fire → stop and revert configuration.
- **Rollback/recovery:** remove/set the line to `false` — returns to the `T6-01` as-is state. No code, no data loss.
- **Downstream tasks:** T6-22 (canonical ledger must be populated before canonical read), T6-28 (§H gate 9).

#### T6-21 — `gba.reconciliation.enabled` ON (soak instrument precondition)

- **Objective:** Turn on the soak instrument so `LEGACY_DRIFT` reports hourly during the window.
- **Decision authority:** **`BD-6-05` via `02B` Decision 2** — *"`gba.reconciliation.enabled` must be turned on as an operational action with gate evidence (G-5) before the window opens"*. `02` §H gate 8 (*"if adopted, `gba.reconciliation.enabled` ON as the instrument"*). `02A` **OI-28 remains `DEFERRED`** — this activation is a **sequencing consequence, not a change to `TD-6-11`**.
- **Exact implementation scope:** Set `FEATURE_GBA_RECONCILIATION_ENABLED=true`, with recorded gate evidence.
- **Files/modules/surfaces:** deploy `.env`; `.env.example:103`; `gba-reconciliation.service.ts:131-136` (scheduled path reads the flag), `:162` (on-demand path is flag-independent).
- **Dependencies:** `T6-01`. **Preconditions:** `T6-01` complete; release window.
- **Required implementation behavior:** Configuration only. **No code commit.** The `DEFERRED` status of `TD-6-11` / `OI-28` is **unchanged** — no activation *rule* is created here.
- **Verification:** flag recorded ON with gate note; an on-demand `runReconciliation(hotelId)` confirms detectors execute (`gba-reconciliation.service.ts:162`); an hourly scheduled run is observed.
- **Evidence:** `E6-21`.
- **Completion criteria:** instrument demonstrably producing hourly reports before the soak window opens.
- **Failure/stop condition:** scheduled runs not observed, or detector `ERROR` at baseline → **the soak may not open** (this would guarantee an F-2 failure).
- **Rollback/recovery:** set flag OFF — scheduled runs stop; on-demand remains available flag-independently. No data written (`automaticRepair: false`).
- **Downstream tasks:** T6-22, T6-23, T6-24.

#### T6-22 — `gba.pickup.canonicalRead` ON + soak open

- **Objective:** Begin the ratified sequence at its first step and open the soak window with its opening evidence.
- **Decision authority:** **`BD-6-05` via `02B` Decision 2**; `CO-6-05`; FDS §28.3/§28.5; P-21.
- **Exact implementation scope:**
  1. Set `FEATURE_GBA_PICKUP_CANONICALREAD=true` (operational, outside a commit).
  2. Record **soak-open evidence**:
     - **SO-3 (open)** — flag census: `canonicalRead` **ON**, `canonicalWrite` **OFF**, `gba.reconciliation.enabled` **ON**, each with its gate evidence.
     - **SO-2 (open)** — read-path parity: legacy-read vs canonical-read pickup responses compared field-by-field for **all hotels and allotments in scope (full set — no sampling)**, equal per the byte-identical contract.
  3. Record the window's opening timestamp.
- **Files/modules/surfaces:** deploy `.env`; `.env.example:105-106`; `allotment.controller.ts:302-313` (`canonicalRead` read at `:313`; byte-identical aliasing contract at `:302-312`).
- **Dependencies:** `T6-20` (canonical ledger populated), `T6-21` (instrument ready). **Preconditions:** both complete; **`canonicalWrite` must be OFF** (it is, per `T6-01` baseline).
- **Required implementation behavior:** Only the **read** leg moves. `canonicalWrite` stays OFF. Never both ON.
- **Verification:** flag census confirms exactly the SO-3 configuration; SO-2 parity computed over the full set and recorded as equal; no write-path change observed.
- **Evidence:** `E6-22`.
- **Completion criteria:** soak window open with `E6-22` recorded.
- **Failure/stop condition:** `canonicalWrite` observed ON → **immediate failure (F-4)**; parity mismatch at open → **F-5**. In either case do not open the window; roll back `canonicalRead` to OFF.
- **Rollback/recovery:** set `canonicalRead` **OFF** → reads return to the legacy source (default path, **no code change**, FDS §24.7). `canonicalWrite` remains OFF, so dual-write continues and **no data is lost**; `G-10` stays gated.
- **Downstream tasks:** T6-23.

#### T6-23 — Soak window execution & monitoring

- **Objective:** Run the 14-day / 336-run soak and detect any failure condition the moment it occurs.
- **Decision authority:** **`BD-6-05` via `02B` Decision 2** — the five finalized parameters, in full.
- **Exact implementation scope:** Maintain the window and record continuously:

  **Duration — parameter 1:** **14 consecutive calendar days AND ≥ 336 consecutive hourly `LEGACY_DRIFT` runs — whichever completes later.** (336 = 14 × 24 at `gba-reconciliation.service.ts:131`.)

  **Threshold — parameter 2:** **Zero — 0 discrepancies.** Exact-match, i.e. the detector's existing implemented behavior (`gba-reconciliation.service.ts:379`). **No tolerance band is introduced.**

  **Required evidence — parameter 3:**
  - **SO-1** — ≥ 336 consecutive hourly `LEGACY_DRIFT` runs, hotel-scoped, each clean with `findingCount: 0`, **no gaps** in the hourly series.
  - **SO-2 (close)** — read-path parity re-run over the **full set** at soak close (the open reading is recorded in T6-22).
  - **SO-3 (close)** — flag census re-recorded at soak close (the open reading is recorded in T6-22).
  - **SO-4** — `canonicalWrite` provably **OFF for the entire window**.

  **Failure conditions — parameter 4:**

  | ID | Failure |
  |---|---|
  | **F-1** | any `LEGACY_DRIFT` run reports `findingCount > 0` |
  | **F-2** | any run reports `ERROR`, `SCHEMA_PENDING`, or compares no rows (**absence of evidence is failure, not success**) |
  | **F-3** | a gap exists in the hourly series (a missed run) |
  | **F-4** | `canonicalWrite` observed ON at any point |
  | **F-5** | SO-2 parity mismatch at soak open or soak close |

  **Restart / rollback — parameter 5:**
  - **Restart:** any failure resets the clock to **zero** — the full 14 days / 336 runs must be re-earned. Root cause recorded **before** the window reopens.
  - **Rollback:** `canonicalRead` **OFF** → legacy read (no code change, FDS §24.7). `canonicalWrite` **remains OFF** → dual-write continues, **no data lost**, `G-10` stays gated.
  - **Forward hold:** `canonicalWrite` may **not** be enabled until a full clean window completes and SO-1…SO-4 are recorded. **Never both ON.**

- **Files/modules/surfaces:** `gba-reconciliation.service.ts:131` (cadence), `:162` (on-demand), `:347-392` (`LEGACY_DRIFT`), `:379` (exact-match); `allotment.controller.ts:302-312` (parity contract); `reservation-pickup-cascade.service.ts:102` (`canonicalWrite` OFF → both ledgers written, i.e. directly comparable).
- **Dependencies:** `T6-22`. **Preconditions:** window open with `E6-22` recorded.
- **Required implementation behavior:** **No code change during the window.** The soak is observation and evidence capture only. The detector is the existing `LEGACY_DRIFT` — **no second reconciliation detector is invented** for this purpose.
- **Verification (status vocabulary — see §11.2):** each hourly report's detectors are `CLEAN` with report-level `findingCount: 0`; the series is gapless; `canonicalWrite` remains OFF throughout.
- **Evidence:** `E6-23` (hourly series + continuity record).
- **Completion criteria:** ≥ 336 consecutive clean hourly runs across ≥ 14 consecutive days, with SO-4 continuity provable.
- **Failure/stop condition:** any of F-1…F-5. **Absence of evidence counts as failure** — no data, a gap, or an `ERROR` is not a pass. On failure: record root cause, reset clock to zero, and either reopen the window or roll back.
- **Rollback/recovery:** `canonicalRead` OFF (operational, no code). If a **code** fault is discovered during the window, stop the soak, fix under a normal gated task, and restart the clock from zero.
- **Downstream tasks:** T6-24.

#### T6-24 — Soak exit verification

- **Objective:** Produce the single pass/fail determination that gates `canonicalWrite`.
- **Decision authority:** **`BD-6-05` via `02B` Decision 2, parameter 5 forward hold.**
- **Exact implementation scope:** Compile and sign off the four evidence items together: **SO-1** (≥ 336 gapless clean runs), **SO-2 close** (full-set parity equal), **SO-3 close** (flag census as required), **SO-4** (`canonicalWrite` never ON). Confirm zero F-1…F-5 occurrences across the whole window.
- **Files/modules/surfaces:** `E6-22`, `E6-23` inputs; output recorded as `E6-24`.
- **Dependencies:** `T6-23`. **Preconditions:** window complete without failure.
- **Required implementation behavior:** Verification and compilation only — **no flag change happens in this task**.
- **Verification:** all four SO items present, each with its own recorded evidence; the failure register shows no F-1…F-5.
- **Evidence:** `E6-24`.
- **Completion criteria:** SO-1…SO-4 all recorded satisfied → T6-25 unlocked.
- **Failure/stop condition:** any SO item missing or any F condition having occurred → **T6-25 may not start**; return to T6-23 restart rules (clock to zero).
- **Rollback/recovery:** as T6-23.
- **Downstream tasks:** T6-25, T6-28 (§H gates 8, 9).

#### T6-25 — `gba.pickup.canonicalWrite` ON + post-cutover verification

- **Objective:** Complete the ratified sequence at its final step, and verify the canonical ledger is now the write authority for pickups.
- **Decision authority:** **`BD-6-05` via `02B` Decision 2**; `CO-6-05`; FDS §28.5; P-21.
- **Exact implementation scope:**
  1. Set `FEATURE_GBA_PICKUP_CANONICALWRITE=true` **only after** `E6-24` records a full pass (operational, outside a commit).
  2. Post-cutover verification: legacy `allotment_pickups` becomes read-only; canonical `group_pickups` is the write authority; `reservation-pickup-cascade.service.ts:102` now reports `legacyReadOnly = true`.
  3. Record final flag census (§H gate 9).
- **Files/modules/surfaces:** deploy `.env`; `.env.example:106`; `reservation-pickup-cascade.service.ts:94-121`; `allotment.controller.ts:313`.
- **Dependencies:** `T6-24` (**mandatory**). **Preconditions:** `E6-24` pass; `canonicalRead` still ON; never both ON at any time.
- **Required implementation behavior:** Configuration only, outside a commit. The sequence may not be altered.
- **Verification:** flag census shows `canonicalRead` ON + `canonicalWrite` ON **only after** soak exit (the post-soak steady state that the two legs implement — `reservation-pickup-cascade.service.ts:94-95`; the prohibition is against enabling `canonicalWrite` **during** the soak and against enabling the two in one step — **see §17 row 4**, which records this reading and its escalation trigger); cascade observed writing canonical only; no legacy counter change.
- **Evidence:** `E6-25`.
- **Completion criteria:** `canonicalWrite` ON with recorded gate evidence and post-cutover verification.
- **Failure/stop condition:** enabling without `E6-24`; enabling inside a commit; observed legacy ledger still being written → stop and roll back.
- **Rollback/recovery:** set `canonicalWrite` **OFF** → dual-ledger write resumes (BLK-3 behaviour), `canonicalRead` may then also be set OFF to return fully to the legacy path. **No data is lost** on rollback, because dual-write was the standing state (`02B` Decision 2, parameter 5).
- **Downstream tasks:** T6-26, T6-28 (§H gate 9).

#### T6-26 — Legacy `allotment_pickups` removal (G-10)

- **Objective:** Execute the `G-10` disposition now that its gate has opened.
- **Decision authority:** **`02` §I `G-10` = `KEEP → REMOVE after gba.pickup.canonicalWrite ON`**; `BD-6-05` (the sequence that opens the gate); `01` G-10.
- **Exact implementation scope:** Remove the legacy `allotment_pickups` ledger **only after** `canonicalWrite` is ON with recorded soak evidence. Scope is the legacy ledger's read/write surface — `allotment.controller.ts:338,346,379` legacy branch and `reservation-pickup-cascade.service.ts:113` legacy write.
- **Files/modules/surfaces:** `packages/db/schema.prisma:137 model allotment_pickup` (**the model is not dropped — no schema change is authorised, `L-r-23`**; removal here means removal of the *read/write surface*, not a table drop); `allotment.controller.ts:338-379`; `reservation-pickup-cascade.service.ts:113`.
- **Dependencies:** `T6-25`. **Preconditions:** `E6-25` recorded; `E6-24` recorded pass.
- **Required implementation behavior:** **Gated removal.** This task's execution status is `GATED` until `canonicalWrite` ON is evidenced. Table deletion remains **Phase 11** alongside `G-1` (`L-r-03`, P-16).
- **Verification:** legacy ledger no longer read or written by application code; canonical path unaffected; `t66-gba-reconciliation.*` `LEGACY_DRIFT` detector behaviour re-checked (if the legacy source is removed, the detector's scope must be re-evidenced rather than silently left reading nothing — see failure condition).
- **Evidence:** `E6-26`.
- **Completion criteria:** legacy pickup ledger surface removed; canonical read/write unaffected; no table dropped.
- **Failure/stop condition:** if removal would leave `LEGACY_DRIFT` reading an empty source (which would then produce vacuous passes), **stop** and record it — a detector that compares nothing is an F-2-equivalent condition, and silently continuing would corrupt the soak's evidence meaning. Escalate with a decision ID if detector scope needs restating.
- **Rollback/recovery:** revert the commit — the legacy surface returns; `canonicalWrite` can be set OFF independently.
- **Downstream tasks:** T6-28.

---

### WS-8 — Exit Evidence

#### T6-27 — E-8 reconciled executed baselines

- **Objective:** Produce reconciled executed baselines so no baseline is silently redefined and no code-changing exit closes without them.
- **Decision authority:** FDS §31 **E-8**; `02A` **OI-25 = `EXECUTION GATE ONLY`**; `phase-5/04:280` (*"exit blocked until reconciled; baselines never silently redefined"*); `02` §Q.2 item 9.
- **Exact implementation scope:** Execute the full test battery against a DB environment and reconcile executed results against the recorded baselines. Any baseline that changed must be **re-captured explicitly and attributed** — never silently redefined.
- **Files/modules/surfaces:** `apps/api` jest suite (`pnpm --filter api test`); `apps/web` suite (`pnpm --filter web test`); `phase-5/06_EXECUTION_EVIDENCE.md` baseline set; `E6-05` … `E6-19` outputs.
- **Dependencies:** all code-changing tasks — `T6-05`, `T6-06`, `T6-09`, `T6-10`, `T6-12`, `T6-13`, `T6-15`, `T6-17`, `T6-18`, `T6-19`, `T6-26`.
- **Preconditions:** those tasks complete; DB test environment available.
- **Required implementation behavior:** Execute and reconcile. **No baseline may be edited to make a run pass.**
- **Verification:** full suites green; every baseline delta explicitly listed with a cause and a reference to the task that caused it.
- **Evidence:** `E6-27`.
- **Completion criteria:** reconciled executed baselines recorded; `02` §Q.2 item 9 satisfied.
- **Failure/stop condition:** any unexplained baseline change → **blocks every code-changing exit**; stop and investigate. Do not accept a run as passing because a baseline moved.
- **Rollback/recovery:** investigation; if a change caused an unintended regression, revert that change (per its own task rollback).
- **Downstream tasks:** T6-28.

#### T6-28 — §H cutover exit gates 1–9 evidence assembly & Phase 6 completion record

- **Objective:** Assemble the recorded evidence that constitutes Phase 6 completion. **Completion requires recorded evidence, never a statement such as "implemented successfully."**
- **Decision authority:** `02` §H.3 gates 1–9; `CO-6-01` … `CO-6-07`; `02A` §I; `02B` Decisions 1–3 (each names the gate it satisfies).
- **Exact implementation scope:** Index every `E6-nn` entry against the gate it satisfies:

  | §H gate | Requirement | Satisfied by |
  |---|---|---|
  | **1** | E-1 answered (store population + writers identified) | `E6-02`, `E6-03` (T6-02, T6-03) |
  | **2** | E-5 answered **or** a BD-6-03 date set | `E6-04` — **the E-5 branch; no date exists by design** (02B Decision 1) |
  | **3** | TD-6-09 confirmed and implemented with DB-backed proof | `E6-09`, `E6-10` (T6-09, T6-10) |
  | **4** | Counter-vs-balance comparator running report-only with persisted findings | `E6-11`, `E6-13` (T6-11, T6-13) |
  | **5** | Population census produced; `LEGACY` / `ASSERTION_MANAGED` / no-row counts known | `E6-14` (T6-14) |
  | **6** | H-1 generator eliminated — no path mutates `reservations` stay fields without the assertion port | `E6-16`, `E6-17` (T6-16, T6-17) |
  | **7** | TD-6-01 defect fixed, authority surface measurable with a `roomType` filter | `E6-05` (T6-05) |
  | **8** | BD-6-05 soak criteria written; `gba.reconciliation.enabled` ON as the instrument | Criteria: `02B` Decision 2 (this plan §11). Instrument: `E6-21`. Soak: `E6-22`, `E6-23`, `E6-24` |
  | **9** | All flags at ratified states with recorded gate evidence; no flag changed inside a commit | `E6-01`, `E6-20`, `E6-21`, `E6-22`, `E6-25` |

  Plus `CO-6-*` confirmations: `CO-6-01` (no legacy writer removed before E-1 — verified by ordering), `CO-6-02`/`CO-6-03` (census → tooling → first-touch), `CO-6-05` (flag order never altered), `CO-6-06`/`CO-6-07` (flag actions operational, none in a commit).

- **Files/modules/surfaces:** `04_EXECUTION_EVIDENCE.md` (**created when execution begins**, per 02B Decision 4); this plan §14.
- **Dependencies:** all other tasks.
- **Preconditions:** `E6-01` … `E6-27` recorded.
- **Required implementation behavior:** Assembly and attestation only. A gate is satisfied by an **evidence reference**, not by an assertion.
- **Verification:** each of the nine gates maps to at least one `E6-nn`; no gate is marked satisfied without a reference; `02` §Q.2 items 1–9 each map to evidence.
- **Evidence:** `E6-28`.
- **Completion criteria:** all nine §H gates satisfied with recorded evidence → Phase 6 complete.
- **Failure/stop condition:** any gate without an evidence reference → Phase 6 is **not** complete. Record the gap; do not close.
- **Rollback/recovery:** none (assembly only).
- **Downstream tasks:** Phase 11 receives `G-1`, `G-11`, whichever of `G-5` remains KEEP, and the `LEGACY` population rows if T6-15 was built (`02` §R).

---

## 8. Dependency / Execution Order

### 8.1 The two ratified sequences — both must remain intact

**Sequence A — capacity/authority cutover (`DS-01 → DS-03 → DS-05`, FDS §24.2, no step skipped):**

| Step | Ratified precondition | Supplied here by |
|---|---|---|
| **DS-01 operational** | evaluator bound + E-1 + E-3 + parity soak recorded | bindings verified in T6-03; **E-1** = T6-02; **E-3** = T6-03; parity soak = Phase 5 record (already closed — referenced, never re-run) |
| **DS-02** *(frontend cutover)* | DS-01 operational | **NOT in Phase 6 enumerated scope** (`02` §R). Recorded here only so the chain is not misread as authorising DS-02 work. **No DS-02 task exists.** |
| **DS-03 read retirement** | may proceed in parallel (read side) | largely done (engine routes 404 by construction, `t522`–`t525`) |
| **DS-03 booking retirement** | operational authority | **T6-17** (the `engine/modify` retirement = `BD-6-03`) |
| **DS-05 leg removal** | DS-01 **and** DS-03 dispositions landed | repository leg already removed by `T5-50` (verified, not re-done); external leg = **T6-18** |

**Sequence B — pickup ledger cutover (`canonicalRead → soak → canonicalWrite`, FDS §28.3/§28.5):**

`T6-20` (cascade ON) → `T6-21` (reconciler ON) → `T6-22` (canonicalRead ON + soak open) → `T6-23` (soak) → `T6-24` (soak exit) → `T6-25` (canonicalWrite ON) → `T6-26` (G-10 removal)

**Never both ON as a shortcut. Never altered.**

### 8.2 Full dependency graph

```
WS-1   T6-01 ──┬─► T6-02 ─────────────┐
                ├─► T6-03 ─────────────┤
                ├─► T6-04 ─────────────┤
                ├─► T6-20              │
                ├─► T6-21              │
                └─► T6-14 ◄─ T6-02     │
                        │              │
WS-2   T6-05 (indep.)   │              │
        T6-06 (indep.)  │              │
        T6-07 (indep.)  │              │
        T6-08 (indep.)  │              │
                        │              │
WS-3   T6-09 ─► T6-10 ──┼──────────────┤
                        │              │
WS-4   T6-11 ─┬─► T6-12 ◄─ T6-10       │
               └─► T6-13               │
                                        │
WS-5   T6-14 ─► T6-15                   │
                                        │
WS-6   T6-16 ◄─ T6-02, T6-03, T6-04 ───┘
          │
          └─► T6-17 ─► T6-18 ─► T6-19
                (G-15 removal inside T6-17)

WS-7   T6-20 ─┐
        T6-21 ─┼─► T6-22 ─► T6-23 ─► T6-24 ─► T6-25 ─► T6-26
               │
WS-8   T6-27 ◄─┴─ all code-changing tasks
        T6-28 ◄── all tasks
```

### 8.3 Recommended execution order

Phases below are **convenience groupings for a human executor, not ratified stages** (02B Decision 4). Any task marked `READY` may start as soon as its own preconditions hold.

| Order | Tasks | Gate to enter |
|---|---|---|
| **1** | T6-01 | — |
| **2** (parallel) | T6-02, T6-03, T6-04, T6-05, T6-06, T6-07, T6-08, T6-09, T6-11, T6-14 | T6-01 |
| **3** | T6-10, T6-12, T6-13, T6-15 | T6-09 / T6-11 / T6-14 respectively |
| **4** | T6-16 | `E6-02`, `E6-03`, `E6-04` all recorded |
| **5** | T6-17 → T6-18 → T6-19 | RC-1…RC-5 `SATISFIED` |
| **6** (parallel with 4–5) | T6-20, T6-21 | T6-01 |
| **7** | T6-22 → T6-23 → T6-24 → T6-25 | `E6-20`, `E6-21` recorded; then SO-1…SO-4 |
| **8** | T6-26 | `E6-25` recorded |
| **9** | T6-27 | all code-changing tasks complete |
| **10** | T6-28 | `E6-01` … `E6-27` recorded |

### 8.4 Ordering interpretations (documented, not decisions)

**Interpretation 1 — `RC-4` / `TD-6-02` sequencing (required to execute `02B` Decision 1).**

`02B` lists RC-4 (*"`TD-6-02` disposition executed — no path mutates `reservations` stay fields without the assertion port"*) among the **eligibility conditions** for retiring the route, while `02` §H gate 6 treats the same execution as the **outcome** of the retirement. Read literally, the route could only retire after it had already retired.

The plan resolves the ordering **without altering the condition**:

1. **Pre-eligibility (T6-16):** verify that every in-repo path *other than* `POST /rates/engine/modify` mutates `reservations` stay fields only through `RESERVATION_AVAILABILITY_PORT` — repository, check-out, upgrade-room, and holds.
2. **Retirement (T6-17):** remove the last non-conforming path.
3. **Post-verification (T6-16 re-run / T6-17 evidence):** confirm RC-4 in full — **zero** paths mutate stay fields without the port.

All three steps are required. RC-4 is satisfied only after step 3 — exactly when §H gate 6 closes. **The condition is unchanged; only its execution order is made explicit.**

**Interpretation 2 — `gba.reconciliation.enabled` activation.**

`02B` Decision 2 makes activation of this flag a **precondition of the soak**, while `TD-6-11` / `OI-28` remains `DEFERRED` as a *rule*. T6-21 therefore executes a **sequencing consequence already named by `02B`**, and creates no activation rule. `TD-6-11` / `OI-28` status is unchanged.

---

## 9. Reconciliation Plan

### 9.1 Scope

"Reconciliation" here means **dual-source reconciliation closure** (B-1/B-2/B-3) — the legacy/counter representation versus the assertion/balance authority. GBA wash reconciliation (`01` I-4/I-5) is out of scope (BD-6-08 `DEFERRED`).

### 9.2 The ratified boundary (non-negotiable — `02` §G.2)

| # | Invariant |
|---|---|
| 1 | **Report, never repair.** REQ-25.2.1/.4 — discrepancies are flagged facts; balances are never "fixed" |
| 2 | **Legacy never wins.** REQ-25.2.2 — where authority and legacy differ, authority is the domain truth |
| 3 | **Reconciliation is evidence, not authority.** REQ-25.3 — may not feed availability decisions, eligibility, or publication; may not write domain state |
| 4 | **Only one sanctioned comparison** (REQ-25.1); an added comparator must be a further sanctioned, read-only comparison — never a second truth |
| 5 | **Layer discipline.** FDS §26.2 layer 3 — reconciliation evidence only; a report may never become a second business authority |
| 6 | **Availability-state ownership.** FDS §26.3 — only the assertion engine may create/update/delete availability-state rows |
| 7 | **No auto-repair under any flag.** `automaticRepair` stays `false` in every configuration and every future job |

### 9.3 The two comparators

| | **Projection comparator (I-1 — exists)** | **Counter-vs-balance comparator (I-2 — to build)** |
|---|---|---|
| Task | T6-12 | T6-13 |
| **Population** | `(hotel_id, room_type, stay_date)` where a legacy `availability` row exists, within requested scope | union of legacy `availability` rows and `availability_assertion_balances` rows within scope |
| **Legacy / comparison source** | `availability.available` (observed, `authoritative: false`) | `availability.reserved` (`schema.prisma:439`) |
| **Authority / expected source** | canonical `sellableAvailable` from the snapshot (`availability-reconciliation.service.ts:22`) | `availability_assertion_balances.asserted_quantity` (`schema.prisma:17320`) |
| **Status as-built** | LIVE, on-demand, no cron, no store (`01` I-1, I-7, N-4) | **ABSENT** (`01` I-2) |
| **Classification** | `MATCH` · `DIAGNOSTIC_VARIANCE` · `MISSING_PROJECTION` · `UNRESOLVED_SOURCE` (existing, unchanged) | `MATCH` · `COUNTER_VARIANCE` · `MISSING_LEGACY_COUNTER` · `MISSING_BALANCE` · `UNCOMPARABLE` |
| **Expected/result interpretation** | `observed === expected` ⇒ `MATCH`. `VARIANCE` is **projection-drift evidence** (REQ-25.1) — valid for its ratified purpose. Post-T6-10, `expected` reflects assertion balances. | Equality ⇒ the two representations of reservation commitment agree. Inequality ⇒ magnitude of the `H-2` divergence, as a **flagged fact only**. |
| **Cadence** | hourly, flag default OFF (`02A` OI-14 `CLOSED`) | same mechanism as T6-12 |
| **Repair** | `automaticRepair: false` — literal, always | identical |
| **Evidence entry** | `E6-12` | `E6-13` |

**Instrument reuse.** The `LEGACY_DRIFT` detector in `gba-reconciliation.service.ts:347-392` is the established instrument for the **pickup-ledger** soak and is used unchanged for that purpose (§11). **No second reconciliation detector is invented merely because this plan needs a task.**

### 9.4 Documented discrepancy — I-2 register wording

`01` §I I-2 and `02` §G.1 name the comparison as **`availability.available` vs `availability_assertion_balances.asserted_quantity`**. Those two columns are not commensurate: `available` is a *remaining* quantity while `asserted_quantity` is a *consumed* quantity. Comparing them directly would additionally duplicate `I-1`, which already compares `available` against the authority's remaining figure.

**Plan treatment, under the §2.1 conflict rule:** the finalized *decision* — **build I-2, report-only, in scope, sanctioned** — is preserved exactly. The **quantity pairing** is set to the commensurate pair, `availability.reserved` ↔ `asserted_quantity`, which is what "counter-vs-balance" denotes against these two schemas. This is a targeting detail resolved at tier F (workspace schema), recorded here rather than applied silently. **The decision is not changed.** If the executor judges the register wording to be binding instead, stop T6-13 and escalate for a decision ID rather than implementing an arithmetic that cannot hold.

### 9.5 Repeated/idempotent execution

Both comparators are **read-only and therefore trivially repeatable**: re-running produces a new report and never mutates state. Findings are keyed by run so a re-run cannot overwrite or "repair" a prior finding. Population census (T6-02, T6-14) is likewise read-only and repeatable. The only non-idempotent write in the plan is T6-15's assignment tool, which is explicitly required to be idempotent and is verified as such.

### 9.6 Remediation paths — what is and is not authorised

| Authorised remediation | Task | Authority |
|---|---|---|
| Retire the `H-1` generator (the external modify path) | T6-17 | BD-6-03, TD-6-02 |
| First-touch / bounded-batch population conversion | T6-15 | TD-6-04, D-21 |
| Human investigation recorded as evidence | recorded in `04_EXECUTION_EVIDENCE.md` | REQ-25.3 |

**Not authorised under any circumstance:** automatic repair; balance "fixing"; reconciliation output feeding an availability decision, eligibility, or publication; legacy winning a divergence; bulk backfill. *(REQ-25.2, REQ-25.3, D-21, `02` §G.2.)*

### 9.7 Stop / fail conditions (reconciliation)

Stop immediately and revert on: any write statement reachable from a reconciler; `automaticRepair` not literally `false`; any consumer reading findings to make an availability decision; any schema/table authoring attempt; any baseline edited to make a run pass.

---

## 10. Cutover Plan

The cutover is expressed as **seven distinct phases with explicit entry and exit criteria.** It is never a single deployment action.

### Phase 1 — Preparation

| Field | Value |
|---|---|
| **Entry criteria** | T6-01 complete (as-is flag census recorded) |
| **Tasks** | T6-02 (E-1), T6-03 (E-3), T6-04 (E-5), T6-05, T6-06, T6-07, T6-08, T6-09, T6-10, T6-11, T6-12, T6-13, T6-14, T6-15 |
| **Exit criteria** | `E6-02`, `E6-03`, `E6-04` recorded; read contract implemented and proven (`E6-09`, `E6-10`); both reconcilers scheduled and report-only (`E6-12`, `E6-13`); population census produced (`E6-14`) |
| **Rollback** | Revert individual commits; no flag or data state changed in this phase |

### Phase 2 — Canonical read

| Field | Value |
|---|---|
| **Entry criteria** | Phase 1 exit; `E6-20` (cascade ON) and `E6-21` (reconciler ON) recorded; **`canonicalWrite` confirmed OFF** |
| **Tasks** | T6-22 |
| **Action** | Set `gba.pickup.canonicalRead` **ON** — read leg only |
| **Exit criteria** | SO-3 (open) flag census recorded; SO-2 (open) full-set parity equal; window timestamped; `E6-22` recorded |
| **Rollback** | `canonicalRead` **OFF** → legacy read (default, no code, FDS §24.7) |

### Phase 3 — Required verification (at soak open)

| Field | Value |
|---|---|
| **Entry criteria** | Phase 2 complete |
| **Tasks** | verification embedded in T6-22 and re-run at T6-24 |
| **Checks** | SO-2 open parity (full set, no sampling) · SO-3 open flag census (`canonicalRead` ON, `canonicalWrite` OFF, `gba.reconciliation.enabled` ON) · instrument producing hourly reports |
| **Exit criteria** | all three recorded equal/correct |
| **Fail path** | parity mismatch ⇒ **F-5**; `canonicalWrite` observed ON ⇒ **F-4** → roll back `canonicalRead`, do not open the window |

### Phase 4 — Soak

| Field | Value |
|---|---|
| **Entry criteria** | Phase 3 exit with `E6-22` recorded |
| **Tasks** | T6-23 (run), T6-24 (exit determination) |
| **Duration** | 14 consecutive calendar days **AND** ≥ 336 consecutive hourly runs — **whichever completes later** |
| **Threshold** | **Zero** (exact-match, existing detector behavior) |
| **Exit criteria** | SO-1 (≥ 336 gapless clean runs), SO-2 (close parity), SO-3 (close census), SO-4 (`canonicalWrite` OFF throughout) — all recorded, zero F-1…F-5 |
| **Rollback** | `canonicalRead` OFF (no code, no data loss); `canonicalWrite` stays OFF; clock resets to zero on any failure |

### Phase 5 — Canonical write

| Field | Value |
|---|---|
| **Entry criteria** | **`E6-24` records a complete pass.** This gate is absolute |
| **Tasks** | T6-25 |
| **Action** | Set `gba.pickup.canonicalWrite` **ON**, outside any commit, with gate evidence |
| **Exit criteria** | legacy `allotment_pickups` read-only; canonical `group_pickups` write authority; `E6-25` recorded; final flag census recorded |
| **Rollback** | `canonicalWrite` OFF → dual-ledger write resumes; `canonicalRead` OFF → full return to legacy. **No data lost** |

### Phase 6 — Post-cutover verification

| Field | Value |
|---|---|
| **Entry criteria** | Phase 5 exit |
| **Tasks** | T6-25 verification, T6-27 (E-8 baselines) |
| **Checks** | cascade writes canonical only · no legacy counter change · executed baselines reconciled against prior baselines with every delta attributed · `t66`, `t557`, `t561`, `t528` pins green |
| **Exit criteria** | `E6-25`, `E6-27` recorded |
| **Fail path** | unexplained baseline delta ⇒ **stop**; revert the causing change; never edit a baseline to pass |

### Phase 7 — Legacy retirement eligibility & execution

| Field | Value |
|---|---|
| **Entry criteria** | RC-1…RC-5 all `SATISFIED` (`E6-16`), which requires Phase 1's `E-1`/`E-3`/`E-5` evidence |
| **Tasks** | T6-16 (eligibility) → T6-17 (route retirement, `G-15`) → T6-18 (`G-2`) → T6-19 (`G-16`).

| **Exit criteria** | `E6-16` shows RC-1…RC-5 all `SATISFIED`; `E6-17` shows the route returning 404 and a zero-caller scan; `E6-18` shows `InventoryDomainService` gone with zero production `UPDATE availability` statements; `E6-19` shows dead DI removed and the scan-gate pin updated; `02` §H gate 6 and `02` §Q.2 item 6 satisfied |
| **Rollback** | revert the commits — route, service, dead DI and pins return (code-only; `L-r-15`, no schema or data change). **Rollback restores the `H-1` generator**, so `02B` Decision 1 eligibility must be re-recorded by a fresh T6-16 run before any further retirement attempt |
| **Fail path** | any residual caller, any remaining path mutating `reservations` stay fields without the assertion port, or any RC found `NOT SATISFIED` after removal → **stop and revert**; the cause is escalated for a new decision ID. **No fallback and no date are pre-authorised** (02B rejected Options B and C) |

> **Phase independence.** Phase 7 depends on Phase 1's `E-1` / `E-3` / `E-5` evidence (`E6-02`, `E6-03`, `E6-04`) and on nothing else from Phases 2–6; Phases 2–6 (the pickup-ledger cutover) depend on nothing from Phase 7. The seven phases are a convenience partition for a human executor, **not a stage model** (§8.3; `02B` Decision 4).

---

## 11. Canonical Cutover — Soak Gates

Authority: **`BD-6-05` as finalized by `02B` Decision 2 (OI-04)** — all five parameters set. Supporting: `CO-6-05`, FDS §28.3 / §28.5, AC-39, P-21, `L-r-05`, `L-r-16`.

### 11.1 Ratified sequence and gate linkage

`gba.consumers.cascade` ON → `gba.reconciliation.enabled` ON → `gba.pickup.canonicalRead` ON → **soak** → `gba.pickup.canonicalWrite` ON → `G-10` removal. The order is never altered; each step is an `OPERATIONAL` flag action performed outside any commit with recorded gate evidence (`L-r-16`, G-5).

| Step | Task | Evidence | Gate it serves |
|---|---|---|---|
| Cascade ON — deploy invariant (`BD-6-04`) | T6-20 | `E6-20` | `02` §Q.2 item 7; §H gate 9 |
| Reconciler ON — soak instrument (`BD-6-05` consequence; `OI-28` still `DEFERRED`) | T6-21 | `E6-21` | `02` §H gate 8 (instrument half) |
| `canonicalRead` ON + soak open — *the second of the "two operational flag actions"* named by `02B` | T6-22 | `E6-22` | soak SO-2/SO-3 open readings |
| Soak window | T6-23 | `E6-23` | soak SO-1, SO-4 |
| Soak exit determination | T6-24 | `E6-24` | forward hold before `canonicalWrite` |
| `canonicalWrite` ON | T6-25 | `E6-25` | `02` §H gate 9; unlocks T6-26 |
| `G-10` legacy surface removal | T6-26 | `E6-26` | `02` §I `G-10` disposition |

`02B` names **"two operational flag actions + gate evidence"** for *opening the window* — those are T6-21 and T6-22. T6-20 (cascade) is `BD-6-04`'s own deploy invariant and is sequenced first because cascade ON is what begins writing the canonical ledger the soak then compares (`02B` Decision 2, downstream note).

### 11.2 The five finalized parameters, the SO evidence, and the status vocabulary

Reproduced from `02B` Decision 2; nothing here is re-derived or softened.

| # | Parameter | Value |
|---|---|---|
| **1 — duration** | 14 consecutive calendar days **AND ≥ 336 consecutive hourly `LEGACY_DRIFT` runs — whichever completes later.** (336 = 14 × 24 at the existing hourly cadence, `gba-reconciliation.service.ts:131`.) |
| **2 — threshold** | **Zero — 0 discrepancies.** Exact-match, i.e. the detector's existing implemented behavior (`gba-reconciliation.service.ts:379`). **No tolerance band is introduced.** |
| **3 — evidence** | **SO-1…SO-4**, all recorded in `04_EXECUTION_EVIDENCE.md` (§14): **SO-1** ≥ 336 consecutive hourly `LEGACY_DRIFT` runs, hotel-scoped, each `status: OK` and `findingCount: 0`, with **no gaps** in the hourly series. **SO-2** read-path parity — legacy-read vs canonical-read pickup responses compared field-by-field for **all hotels and allotments in scope (full set — no sampling)** at soak open and again at soak close, equal per the byte-identical contract at `allotment.controller.ts:302-312`. **SO-3** flag census at soak open and soak close as operational-action evidence (G-5): `canonicalRead` **ON**, `canonicalWrite` **OFF**, `gba.reconciliation.enabled` **ON** — each with its gate evidence. **SO-4** `canonicalWrite` provably **OFF for the entire window**. |
| **4 — failure** | **F-1…F-5** (§11.3). **Absence of evidence is failure, not success.** |
| **5 — restart / rollback / forward hold** | **Restart:** any failure resets the clock to **zero** — the full 14 days / 336 runs must be re-earned; root cause recorded before the window reopens. **Rollback:** `gba.pickup.canonicalRead` **OFF** → reads return to the legacy source (default path, no code, FDS §24.7); `canonicalWrite` **remains OFF**, so dual-write continues, **no data is lost**, `G-10` stays gated. **Forward hold:** `canonicalWrite` may **not** be enabled until a full clean window completes and SO-1…SO-4 are recorded. **Never both ON (FDS §28.5).** |

**Status vocabulary — documented reading (the third §2.1 instance).** `02B` specifies SO-1 as each run reporting `status: OK`. The implemented report vocabulary has no `OK` value: `GbaDetectorStatus = 'CLEAN' | 'FLAGGED' | 'PENDING' | 'ERROR'` (`gba-reconciliation.service.ts:39`), and detector status is assigned `FLAGGED` on findings, `PENDING` when a detector could not run, else `CLEAN` (`:195`), with `ERROR` on exception (`:200`).

**Plan treatment, under the §2.1 conflict rule:** the finalized *decision* — ≥ 336 consecutive clean hourly runs, gapless, `findingCount: 0` — is preserved exactly. For execution, `status: OK` is read as **every detector `CLEAN` with report-level `findingCount: 0`** (and the `SCHEMA_PENDING` detector `CLEAN`, i.e. it ran and compared rows). `PENDING`/`ERROR` and any run that compares no rows satisfy F-2 and therefore fail. This is a vocabulary mapping against the implemented type, recorded here rather than applied silently; **the decision is not changed.** If an executor reads `OK` as a literal string that must appear in stored evidence, stop T6-23 and escalate for a decision ID rather than editing the detector's vocabulary.

### 11.3 Failure conditions — F-1 … F-5 (verbatim)

The soak fails if **any** of the following occurs during the window:

| ID | Failure |
|---|---|
| **F-1** | any `LEGACY_DRIFT` run reports `findingCount > 0` |
| **F-2** | any run reports `ERROR` status, `SCHEMA_PENDING`, or compares no rows (**absence of evidence is failure, not success**) |
| **F-3** | a gap exists in the hourly series (a missed run) |
| **F-4** | `canonicalWrite` is observed ON at any point |
| **F-5** | SO-2 parity mismatch at soak open or soak close |

Under the §11.2 vocabulary mapping, F-2's `SCHEMA_PENDING` means the `SCHEMA_PENDING` detector is not `CLEAN` (`PENDING`, `ERROR`, or returning no compared rows). A detector that compares nothing cannot produce a pass — see also T6-26's stop condition.

### 11.4 Restart, rollback and forward hold

- **Restart:** clock to zero; root cause recorded in `04_EXECUTION_EVIDENCE.md` before reopening; the full 14 days / 336 runs are re-earned (`E6-23` restarts).
- **Rollback:** `canonicalRead` **OFF** (operational, no code, FDS §24.7); `canonicalWrite` stays **OFF**; dual-write continues; **no data lost**; `G-10` stays gated. If a **code** fault is found during the window, stop the soak, fix it under a normal gated task (T6-27 will attribute any baseline delta), and restart the clock from zero.
- **Forward hold:** `canonicalWrite` requires `E6-24` recording a complete pass — SO-1…SO-4 all present, zero F-1…F-5. **Absence of any SO item is a failure, not a delay.**
- **Never both ON.** The prohibition is quoted verbatim in §11.2 and §13.2. §17 row 4 records the one textual ambiguity in that rule and the escalation trigger it carries; nothing in this plan flips either flag outside its task.

### 11.5 What the gates unlock

A clean exit (`E6-24`) unlocks **T6-25** (`canonicalWrite` ON) and, through it, **T6-26** (`G-10`). It also satisfies the execution half of `02` §H gate 8 (the *criteria* half is already satisfied by `02B` itself, per `02B` Decision 2 downstream note) and contributes to gate 9. A failed soak unlocks nothing: `G-10` remains `KEEP → REMOVE after canonicalWrite ON` and stays unexecuted.

---

## 12. Legacy Retirement Plan

Authority: **`02B` Decision 1 (OI-03 / `BD-6-03`) — Option A, gate-based end-of-life, no calendar date**; `02` §I dispositions; `TD-6-02`, `TD-6-13`, `CO-6-01`; FDS §24.3 / §24.4.

### 12.1 Eligibility — RC-1 … RC-5 (no date, no fallback)

| # | Condition (as finalized) | Evidence | Verified by |
|---|---|---|---|
| **RC-1** | E-1 census complete — per-hotel row counts, recency, and writer identification recorded for all six store families | `E6-02` | T6-02 |
| **RC-2** | E-5 answered — external consumer inventory returned and recorded; if an external consumer exists it is re-pointed first; the route becomes eligible only when the **last** external consumer is gone | `E6-04` | T6-04 |
| **RC-3** | DS-01 operational — evaluator bound + E-1 + E-3 + parity soak recorded | `E6-02`, `E6-03`; bindings `availability.module.ts:45,51`; Phase 5 parity soak record | T6-02, T6-03 |
| **RC-4** | `TD-6-02` disposition executed — no path mutates `reservations` stay fields without the assertion port | see §8.4 ordering interpretation | T6-16 (pre), T6-17 (post) |
| **RC-5** | In-repo consumers = ∅ | `rg -n "engine/modify" apps/api/src apps/web` → route definition only, re-run at execution | T6-16 |

- **All five must be recorded `SATISFIED` in `E6-16` before T6-17 starts.** Any `NOT SATISFIED` blocks the retirement.
- **If a condition cannot be met, that is a new escalation requiring its own decision ID** — nothing enters Phase 6 without one. **No fallback is pre-authorised and no calendar date may be set** (`02B` rejected Options B and C: a deprecation window and a fixed date).
- Retirement is **removal**, not deprecation and not hardening-then-leaving: the route returns **404** and the writer is deleted (FDS §24.4; `02B` Decision 1).

### 12.2 Ordering interpretation (pointer, not a decision)

RC-4 and `02` §H gate 6 describe the same fact from opposite sides (condition vs outcome). §8.4 Interpretation 1 records the three-step execution order that satisfies both without altering the condition: **pre-eligibility scan → retirement → full post-verification.** All three steps are mandatory.

### 12.3 `G` dispositions — preserved exactly (`02` §I, confirmed)

**No item is removed by this document.** `KEEP` / `DEFER` / `REMOVE` below are transcribed from `02` §I; the *Phase 6 action* column says what this plan actually schedules.

| Ref | Artefact | Disposition (`02` §I) | Phase 6 action here |
|---|---|---|---|
| **G-1** | legacy `availability` table | **KEEP** read-only → Phase 11 | none — not touched (P-16, `L-r-03`, `L-r-23`) |
| **G-2** | `InventoryDomainService` | **REMOVE** (gated by BD-6-03) | **T6-18** |
| **G-3** | `crs.modifyReservation` + `POST engine/modify` | **DEFER (E-5) → REMOVE** | **T6-04** (E-5) → **T6-16** (eligibility) → **T6-17** (retirement) |
| **G-4** | `evaluateRestrictions` | **DEFER** — keep until E-1 (TD-6-05) | none — out of enumerated scope |
| **G-5** | `room_inventory` | **KEEP read-only *or* REMOVE — undecided** | none — owner choice remains with `BD-6-01`; no task, no implied resolution |
| **G-6** | `channel_restrictions` | **KEEP** (excluded; no writer) | none |
| **G-7** | `restrictions` (CUTOFF) | **KEEP** (excluded; E-4 semantics undefined) | none |
| **G-8** | `rate_restrictions` | **DEFER** — E-1 gate | none — owner decision after E-1 (`BD-6-02` / `TD-6-05`) |
| **G-9** | `channel_availability` / `channel_availability_log` | **KEEP** (out of scope; must be fenced) | none — fencing preserved (§4.2) |
| **G-10** | legacy `allotment_pickups` ledger | **KEEP → REMOVE after `canonicalWrite` ON** | **T6-26**, gated by T6-25 / `E6-25` |
| **G-11** | orphaned FO / guest-profile adapters | **DEFER** — Phase 11 disposal | none |
| **G-12** | `PrismaInventoryReservationAdapter` | **KEEP** (F-03/F-27 pin) | none — do not delete |
| **G-13** | `UnresolvedRestrictionAdapter` | **KEEP** (T5-09 rollback artifact, `availability.module.ts:50`) | none — do not delete until the rollback window closes |
| **G-14** | `CLIENT_MATH_ALLOWED` / WS-N allowlists | **REMOVE entries as each site retires** | rides with each retirement task (`TD-6-13`); no standalone task |
| **G-15** | retired-route delay allowlist (`page.tsx:140`, `use-crs-book.ts:91`, `front-office.api.ts:193`) | **DEFER** until engine routes retire | **inside T6-17** |
| **G-16** | dead `this.crs` / `inventoryDomain` DI in the repository | **REMOVE** after F-5 retires | **T6-19** |

**Also confirmed out of Phase 6:** deletion of the legacy `availability` table (P-16 → Phase 11); admin/mobile availability surfaces (P-19 / M-7, K-18 / K-19 = 0 files).

### 12.4 What this plan must never do to legacy state

1. Never drop, truncate or migrate a table (`L-r-23`; schema work is Phase 2b/11, never here).
2. Never delete `G-1`, `G-6`, `G-7`, `G-9`, `G-12`, `G-13` — these are `KEEP` pins, several of them deliberate rollback artefacts.
3. Never remove a legacy writer before `E-1` closes (`CO-6-01`, `L-r-12`); T6-17 is gated by `E6-16`, which requires `E6-02`.
4. Never treat a retirement as evidence of reconciliation: removal of the `H-1` generator (`02` §H gate 6) is a cutover gate, not a reconciliation result (`§9`).
5. Never retire anything that `02` §I still marks `DEFERRED` without the decision that un-gates it.

## 13. Feature Flag & Rollback Matrix

Authority: FDS §28 (§28.1 inventory, §28.2 mechanism/documentation, §28.3 ON gate, §28.4 what flags must not control, §28.5 preserved rollout invariants); `L-r-05`, `L-r-15`, `L-r-16`; `CO-6-05`, `CO-6-06`, `CO-6-07`; `02` §H gates 8–9; `02B` Decisions 1–2; `01` J-AUD-01.

### 13.1 Flag inventory — as-is, target, task, evidence

All seven flags are env `FEATURE_*`, read via `ConfigService.getFeatureFlag`, **default OFF in code**. `.env` currently carries exactly one flag line: `FEATURE_GBA_A3_AUTHORITATIVE=true` (`.env:68`); the other six are absent, i.e. OFF.

| Flag | Selects | As-is at entry (`T6-01` baseline) | Phase 6 target | Task | Evidence |
|---|---|---|---|---|---|
| `FEATURE_GBA_A3_AUTHORITATIVE` | matrix/snapshot source: authority vs labelled legacy view (FDS §28.3) | **ON** (`.env:68`; `J-AUD-01`) | **unchanged** — may be flipped **OFF only as rollback** (`CO-6-06`, §24.7) | T6-01, T6-03 | `E6-01`, `E6-03` |
| `FEATURE_GBA_CONSUMERS_CASCADE` | GBA consumer cascade writes the canonical pickup ledger | **absent → OFF** | **ON** — deploy invariant (`BD-6-04`, P-21, `.env.example:101`), operational, outside a commit | **T6-20** | `E6-20` |
| `FEATURE_GBA_WASH_SCHEDULERENABLED` | GBA wash scheduler | absent → OFF | **unchanged OFF** — blocked by Deviations A+B (FDS §32) | none | — |
| `FEATURE_GBA_RECONCILIATION_ENABLED` | hourly scheduled `LEGACY_DRIFT` runs (`gba-reconciliation.service.ts:131`) | absent → OFF | **ON before the soak window opens** — soak instrument (`02B` Decision 2); operational, gate evidence | **T6-21** | `E6-21` |
| `FEATURE_GBA_PICKUP_TWOLAYERCONSULT` | two-layer pickup consult | absent → OFF | **unchanged OFF** (`OI-28` / `TD-6-11` remain `DEFERRED`) | none | — |
| `FEATURE_GBA_PICKUP_CANONICALREAD` | pickup read source: canonical vs legacy (`allotment.controller.ts:313`) | absent → OFF | **ON** at soak open; **OFF** on rollback | **T6-22** | `E6-22` |
| `FEATURE_GBA_PICKUP_CANONICALWRITE` | pickup write authority; legacy ledger read-only (`reservation-pickup-cascade.service.ts:102`) | absent → OFF | **ON only after `E6-24` pass**; **OFF** on rollback | **T6-25** | `E6-25` |

**No flag is created by this plan.** `02B`: *"No new tool, table, flag, or schema is created by this decision."* FDS §28.5: Phase 5 added no flag; Phase 6 adds none.

### 13.2 Ratified flag rules (binding on every task above)

1. **Default OFF; `gba.consumers.cascade` MUST be ON at deploy; wash stays blocked** (`L-r-05` — P-21, `.env.example:101`).
2. **No flag flip inside a code commit; every flag change carries recorded gate evidence** (`L-r-16` — G-5, FDS §24.8.4, §28.2). Flag actions are `OPERATIONAL` and are recorded in `04_EXECUTION_EVIDENCE.md` (§14).
3. **Rollback is flag OFF or one binding line; no schema change is required or permitted** (`L-r-15` — FDS §24.7).
4. **Flags select sources/paths, never truth**: no rule, eligibility semantic, or business behaviour changes state based on a flag (FDS §28.4); flags never substitute for executed test evidence (BR-5-033, FDS §28.4).
5. **`gba.a3.authoritative` may be flipped only as a rollback** during Phase 6 (`CO-6-06`); any forward change would require the four BR-5-037 conditions with gate evidence (§28.3) — **this plan proposes no forward change**.
6. **A flag flip is never itself this document's work product** (`CO-6-07`): each flip is an operational step inside its own gated task.
7. **Order:** `canonicalRead` → soak → `canonicalWrite`. **Never altered; never both at once** (`CO-6-05`, `02B` Decision 2 parameter 5 — quoted verbatim in §11.2; see §17 row 4).

### 13.3 Rollback matrix

| Flag taken to OFF | Restores | Data effect | Code change |
|---|---|---|---|
| `gba.a3.authoritative` | labelled legacy matrix (FDS §24.7) | none | none |
| `gba.consumers.cascade` | cascade stops; canonical ledger stops being populated (returns to the `T6-01` as-is state) | none written is lost | none |
| `gba.reconciliation.enabled` | hourly scheduled runs stop; on-demand `runReconciliation(hotelId)` remains available flag-independently (`:162`) | none — `automaticRepair: false`, report-only | none |
| `gba.pickup.canonicalRead` | legacy read, the default path | none — `canonicalWrite` stays OFF so dual-write continues; **no data lost**; `G-10` stays gated | none |
| `gba.pickup.canonicalWrite` | dual-ledger write resumes (BLK-3 standing behaviour); `canonicalRead` may then go OFF for a full return to legacy | none — dual-write was the standing state; **no data lost** | none |
| wash / twoLayerConsult | never changed | — | none |

**Rollback is always operational, never code** (`L-r-15`, `L-r-16`). The single exception is a *code* task revert (T6-05…T6-19, T6-26), which is a normal commit revert; reverting T6-17 additionally **restores the `H-1` generator** and therefore re-opens `02B` Decision 1 eligibility (§12.1).

### 13.4 Deferred, untouched and posture-sensitive flags

- **`OI-28` / `TD-6-11` — activation *rule* for `gba.reconciliation.enabled` and `gba.pickup.twoLayerConsult`: remains `DEFERRED`.** T6-21 executes a **sequencing consequence already named by `02B`** (instrument must be ON before the window); it creates no activation order, no rule and no status change (§8.4 Interpretation 2). `twoLayerConsult` is never touched.
- **`gba.wash.schedulerEnabled`** stays blocked by Deviations A+B (FDS §32); no task enables or modifies it.
- **`J-AUD-01` posture.** Phase 6 starts from an **as-is** flag census (only `gba.a3.authoritative` ON), **not** from any documented end-state. `T6-01` records what is actually configured before anything changes.
- **Nothing outside §13.1's seven flags** is introduced, renamed, relocated into the platform DB, or made to control business rules (FDS §28.2: single mechanism, documented; AC-37).

---

## 14. Evidence Register (`E6-01` … `E6-28`)

Authority: `02B` Decision 4 (OI-10) — Option B; FDS §31 **E-8**; `02A` **OI-25 = `EXECUTION GATE ONLY`** and §K.1 two-tier baseline rule; `02` §Q.2; `02` §H gates 1–9.

### 14.1 Protocol

1. **`04_EXECUTION_EVIDENCE.md` is authorised by OI-10 and is created only when execution begins.** It is **not** created by this plan. When created, it holds exactly the entries below, each with a timestamp, the actor, and the artifact (report, scan output, flag census, test run) it refers to.
2. **Evidence is a record, never an assertion.** "Implemented successfully", "looks correct", or an unrecorded check satisfies no gate. Every `02` §H gate maps to at least one `E6-nn` reference (§16.1); an unreferenced gate is **not** satisfied.
3. **Absence of evidence is failure** (`02B` Decision 2 parameter 4; F-2) for soak evidence, and a **blocker** for any code-changing exit (`OI-25`, E-8).
4. **Flag gate evidence** (`L-r-16`, G-5): every entry in §13.1's Evidence column is recorded *outside any commit* and names the gate it discharges.
5. **Anchor re-verification** (`TD-6-12`): every task re-verifies its own `path:line` anchors at execution and records drift in its `E6-nn` entry instead of assuming the anchors in this plan are current (§2.3, §7 standing duty).
6. **Baseline discipline** (`E6-27`): executed results are reconciled against `phase-5/06_EXECUTION_EVIDENCE.md` baselines; a changed baseline is **re-captured explicitly and attributed**, never silently redefined and never edited to make a run pass.
7. **Reuse of namespaces is forbidden:** no `E-1…E-8`, `S3R-nnn` or `T5-nn` ID is reissued; `E-1…E-8` appear only as *dependencies* (standing prohibition 5).

### 14.2 Register

| ID | Task | What is recorded | Serves |
|---|---|---|---|
| `E6-01` | T6-01 | as-is flag census + tracked-file/git baseline at Phase 6 entry | J-AUD-01; §H gate 9; §Q.2 item 9 |
| `E6-02` | T6-02 | E-1 six-store-family census (per-hotel row counts, recency, writers) | RC-1, RC-3; §H gates 1, 6 |
| `E6-03` | T6-03 | E-3 runtime confirmation of the production DI chain | RC-3; §H gate 1 |
| `E6-04` | T6-04 | E-5 external `/rates/engine/*` consumer inventory | RC-2; §H gate 2 |
| `E6-05` | T6-05 | matrix `roomType` fix + parameterised SQL, **DB-backed** test run | §H gate 7; §Q.2 item 2 |
| `E6-06` | T6-06 | typed delete-reject test run (balances/movements untouched) | §Q.2 item 1 |
| `E6-07` | T6-07 | Phase 3 register annotation + re-verification record | `TD-6-07`, `L-r-25` |
| `E6-08` | T6-08 | FDS §24.1 dated annotation record | `TD-6-12`, D-AUD-02 |
| `E6-09` | T6-09 | additive assertion-balance field + calculator combination | §H gate 3; §Q.2 item 3 |
| `E6-10` | T6-10 | **DB-backed** read-contract proof + `/reconciliation` re-baseline | §H gate 3; `CO-6-04` sequencing rule |
| `E6-11` | T6-11 | findings-persistence mechanism choice + REQ-25.3 boundary demonstration | §Q.2 item 4 |
| `E6-12` | T6-12 | scheduled report-only projection-comparator run (first + recurring) | §H gate 4 |
| `E6-13` | T6-13 | counter-vs-balance comparator run (I-2), report-only | §H gate 4 |
| `E6-14` | T6-14 | reservation population census: `LEGACY` / `ASSERTION_MANAGED` / no-row counts | §H gate 5; §Q.2 item 5 |
| `E6-15` | T6-15 | assignment tooling proof: writes only `reservation_availability_state`, idempotent, hotel-scoped, no balances/movements | §Q.2 item 5; D-21 |
| `E6-16` | T6-16 | RC-1…RC-5 eligibility statement (`SATISFIED` / `NOT SATISFIED` per condition) | `02B` Decision 1 entry gate |
| `E6-17` | T6-17 | 404 proof + zero-caller scan + updated retirement pins | §H gate 6; §Q.2 item 6 |
| `E6-18` | T6-18 | `InventoryDomainService` removal scan; zero production `UPDATE availability` | `02` §I `G-2` |
| `E6-19` | T6-19 | dead DI removed; scan-gate pin updated to record removal | `02` §I `G-16`; `TD-6-13` |
| `E6-20` | T6-20 | `gba.consumers.cascade=true` gate note, changed outside any commit | §Q.2 item 7; §H gate 9 |
| `E6-21` | T6-21 | `gba.reconciliation.enabled=true` gate note + first observed hourly run | §H gate 8 (instrument) |
| `E6-22` | T6-22 | soak-open: SO-3 census (open) + SO-2 parity (open, full set) + window timestamp | soak parameter 3 |
| `E6-23` | T6-23 | SO-1 hourly series (≥ 336, gapless, `findingCount: 0`) + continuity record | soak parameters 1–4 |
| `E6-24` | T6-24 | soak exit determination: SO-1…SO-4 compiled, zero F-1…F-5 | forward hold before `canonicalWrite` |
| `E6-25` | T6-25 | `canonicalWrite=true` gate note + post-cutover verification + final census | §H gate 9 |
| `E6-26` | T6-26 | legacy pickup surface removed; detector scope re-evidenced | `02` §I `G-10` |
| `E6-27` | T6-27 | **E-8** reconciled executed baselines, every delta attributed | FDS §31 E-8; `OI-25`; §Q.2 item 9 |
| `E6-28` | T6-28 | §H gates 1–9 assembly with an evidence reference per gate | Phase 6 completion |

---

## 15. Failure, Stop and Escalation Rules

### 15.1 Universal stop conditions (any task, immediate)

Stop, revert or hold — and record why — on **any** of the following:

1. A **write statement reachable from a reconciler**, or `automaticRepair` not literally `false` (§9.7, REQ-25.2/.3).
2. Reconciliation output read by an availability decision, eligibility computation or publication path (REQ-25.3, `L-r-10`).
3. Any **schema/table/migration authoring** attempt (`L-r-23`), or any table drop (`G-1`, `G-10` model stays until Phase 11).
4. Any **flag changed inside a code commit**, or a flag change without recorded gate evidence (`L-r-16`, G-5).
5. **Both pickup flags ON out of order**, or `canonicalWrite` ON before `E6-24` records a pass (`CO-6-05`, F-4).
6. A **ratified order violated**: `DS-01 → DS-03 → DS-05`, `canonicalRead → soak → canonicalWrite`, or a legacy writer removed before `E-1` (`L-r-11`, `L-r-12`, `CO-6-01`).
7. A **baseline edited to make a run pass**, or an unexplained baseline delta (`E6-27`, `OI-25`).
8. **Scope entry without a decision ID** — anything appearing in source that §1.3/§4 do not already cover (§4.4, standing prohibition 7).
9. A **closed document rewritten** rather than annotated (§24.1/§11.3/Phase 3 register) — `L-r-25`, T6-07/T6-08 stop conditions.
10. Any soak failure condition **F-1…F-5** (§11.3), including a gap or an absence of evidence.
11. A legacy retirement executed with any RC `NOT SATISFIED` (§12.1), or a residual caller after T6-17.
12. Discovery that an `E6-nn` entry would be **claimed without a record** (standing prohibition 1; §14.1 rule 2).

### 15.2 Escalation — the only permitted response to ambiguity

`02B` Decision 1 states the rule this plan applies everywhere: *"If a condition cannot be met, that is a new escalation requiring its own decision ID — nothing enters Phase 6 without one."*

- The executor **may not** resolve ambiguity by choosing the convenient reading, by widening scope, by relaxing a threshold, or by setting a date.
- The task **stops**, the observation is recorded in `04_EXECUTION_EVIDENCE.md`, and a decision is obtained under the §2.1 hierarchy (tier A first).
- **No fallback, no default, no "temporary" state is pre-authorised anywhere in this plan.**

### 15.3 Rollback doctrine

| Situation | Response |
|---|---|
| Flag-caused problem | flag OFF per §13.3 — operational, no code, no data loss |
| Code-caused problem | revert the task's commit; re-run its verification; record the delta in `E6-27` |
| Soak failure | clock to zero; root cause recorded; reopen window or roll back `canonicalRead` (§11.4) |
| Retirement rolled back | the `H-1` generator returns → `02B` Decision 1 eligibility must be re-recorded before another attempt (§12.1) |
| Reconciler misbehaves | flag OFF; on-demand path remains available (`:162`); no data was written (`automaticRepair: false`) |

**No rollback in this plan requires a schema change, a data migration, or a destructive command** (`L-r-15`, `L-r-23`).

### 15.4 What is never done

Automatic repair of balances · legacy winning a divergence · bulk backfill (D-21, `L-r-22`) · reconciliation feeding a domain decision · inventing precedence between conflicting stores (BR-5-003) · guessing an unproven store (BR-5-046) · a new gate namespace or blocker taxonomy · an invented stage, workflow or readiness review · retro-closing or rewriting a closed phase document · deleting a `KEEP`-pinned artefact.

## 16. Completion Criteria

Phase 6 is complete when — and only when — the records below exist. Nothing in this document itself completes any gate.

### 16.1 `02` §H cutover exit gates 1–9 → evidence

| Gate | Requirement (`02` §H.3) | Satisfied by |
|---|---|---|
| **1** | E-1 answered (store population + writers identified) | `E6-02`, `E6-03` (T6-02, T6-03) |
| **2** | E-5 answered **or** a `BD-6-03` date set | `E6-04` — **the E-5 branch; no date exists by design** (`02B` Decision 1) |
| **3** | TD-6-09 confirmed and implemented with DB-backed proof | `E6-09`, `E6-10` (T6-09, T6-10) |
| **4** | Counter-vs-balance comparator running report-only with persisted findings | `E6-11`, `E6-13` (T6-11, T6-13) — plus `E6-12` for the projection comparator |
| **5** | Population census produced; `LEGACY` / `ASSERTION_MANAGED` / no-row counts known | `E6-14` (T6-14) |
| **6** | H-1 generator eliminated — no path mutates `reservations` stay fields without the assertion port | `E6-16`, `E6-17` (T6-16, T6-17) |
| **7** | TD-6-01 defect fixed; authority surface measurable with a `roomType` filter | `E6-05` (T6-05) |
| **8** | BD-6-05 soak criteria written; `gba.reconciliation.enabled` ON as the instrument | criteria: `02B` Decision 2 (§11) — **already written by 02B**; instrument: `E6-21`; soak: `E6-22`, `E6-23`, `E6-24` |
| **9** | All flags at ratified states with recorded gate evidence; no flag changed inside a commit | `E6-01`, `E6-20`, `E6-21`, `E6-22`, `E6-25` (§13) |

Plus `CO-6-*` confirmations: `CO-6-01` (no legacy writer removed before E-1 — enforced by ordering), `CO-6-02`/`CO-6-03` (census → tooling → first-touch), `CO-6-05` (flag order never altered), `CO-6-06`/`CO-6-07` (flag actions operational, none in a commit).

### 16.2 `02` §Q.2 acceptance for the decisions → satisfied by

| §Q.2 | Item | Satisfied by |
|---|---|---|
| 1 | typed deterministic delete-reject; balances/movement rows untouched | `E6-06` (T6-06) |
| 2 | `GET /availability/matrix?roomType=X` succeeds in both flag states; queries 3 and 6 correct with flag OFF | `E6-05` (T6-05) |
| 3 | ASSERTION_MANAGED property reports `sellableAvailable = capacity − Σ balances` on snapshot, matrix and `/reconciliation`; shapes unchanged; snapshot/matrix parity preserved | `E6-09`, `E6-10` (T6-09, T6-10) |
| 4 | scheduled run exists; findings persist; no write statement executes; output never reaches an availability decision | `E6-11`, `E6-12`, `E6-13` (T6-11, T6-12, T6-13) |
| 5 | census produced; assignment tool writes only `reservation_availability_state`, idempotent, hotel-scoped, creates no balances/movements; `LEGACY` refused until first-touch | `E6-14`, `E6-15` (T6-14, T6-15) |
| 6 | no in-repo path mutates `reservations` stay fields without `RESERVATION_AVAILABILITY_PORT`; `rg engine/modify` → zero callers | `E6-17` (T6-17) |
| 7 | `FEATURE_GBA_CONSUMERS_CASCADE=true` recorded with a gate note, changed outside any commit | `E6-20` (T6-20) |
| 8 | written soak exit criterion naming a duration and an evidence threshold; flag flips recorded separately | criteria: `02B` Decision 2 (§11.2); recorded flips: `E6-21`, `E6-22`, `E6-25` |
| 9 | tracked-file git baseline unchanged during the decision phase; every `path:line` anchor re-verified at execution | decision phase: §18 (unchanged); execution: each task's `E6-nn` anchor record (`TD-6-12`), consolidated in `E6-27` |

### 16.3 `02B` Decisions 1–4 → what this plan delivers

| `02B` decision | Owner decision | What remains for implementation | Tasks |
|---|---|---|---|
| **Decision 1** | OI-03 / `BD-6-03` — Option A, gate-based EOL, **no date** | eligibility verification, route retirement (404 + zero-caller scan), `G-2` removal, `G-16` DI removal | T6-16, T6-17, T6-18, T6-19 |
| **Decision 2** | OI-04 / `BD-6-05` — five soak parameters | opening the window (two operational flag actions + gate evidence), running it, recording SO-1…SO-4 | T6-21, T6-22, T6-23, T6-24, T6-25 (→ T6-26) |
| **Decision 3** | OI-07 / `TD-6-09` — Option A read contract | additive field + calculator combination + FDS §11.3 annotation + three DB-backed tests, then `/reconciliation` re-baselining | T6-09, T6-10 |
| **Decision 4** | OI-10 / `TD-6-15` — Option B convention | authoring `03_IMPLEMENTATION_PLAN.md` (**this document**); `04_EXECUTION_EVIDENCE.md` when execution begins | this document; §14 |

### 16.4 Definition of done

1. All 28 tasks' completion criteria met, each with its own `E6-nn` record (§14.2).
2. All nine §H gates satisfied, each with at least one evidence reference (§16.1).
3. All nine §Q.2 acceptance items satisfied (§16.2).
4. Zero open F-1…F-5 conditions; SO-1…SO-4 recorded (§11).
5. `E6-28` assembled — the Phase 6 completion record (T6-28).
6. No gate marked satisfied without a reference; no baseline silently redefined; no document outside this plan modified except by the annotation tasks T6-07 and T6-08.

**Failure of any one item means Phase 6 is not complete.** The gap is recorded; the phase is not closed.

---

## 17. Documented Readings, Source Tensions and Escalation Triggers

This plan quotes finalized decisions rather than paraphrasing them. Where the text of a finalized decision and the text of its authority (or the workspace) do not line up, §2.1 applies: **preserve the finalized decision, document the discrepancy, change nothing.** No row below alters a decision.

| # | Source tension | Where documented | Plan treatment | Decision changed? | Escalation trigger |
|---|---|---|---|---|---|
| **1** | `02` §H gate 6 treats RC-4 as the *outcome* of retiring the route, while `02B` Decision 1 lists RC-4 as an *eligibility condition* — read literally, the route could only retire after it had already retired | §7 (T6-16), §8.4 Interpretation 1, §12.2 | three mandatory steps: pre-eligibility scan of every other path → retirement → full post-verification. All three required; RC-4 satisfied only after step 3 | **No** — condition unchanged, execution order made explicit | any executor asked to retire with RC-4 recorded before step 3 → stop T6-17 |
| **2** | `01` §I I-2 names `availability.available` vs `asserted_quantity`; those columns are not commensurate (`available` is remaining, `asserted_quantity` is consumed), and pairing them would duplicate I-1 | §9.4 | pairing set to the commensurate pair **`availability.reserved` ↔ `asserted_quantity`** (what "counter-vs-balance" denotes against these two schemas); I-2 remains report-only, sanctioned, in scope | **No** — build I-2 report-only; only the quantity pairing is targeted | executor judges the register wording binding → **stop T6-13** and obtain a decision ID |
| **3** | `02B` SO-1 requires `status: OK`; the implemented vocabulary is `'CLEAN' \| 'FLAGGED' \| 'PENDING' \| 'ERROR'` (`gba-reconciliation.service.ts:39`) — there is no `OK` | §11.2 | `status: OK` read as **every detector `CLEAN` + report-level `findingCount: 0`**, hotel-scoped, gapless; `PENDING`/`ERROR`/no-rows fail under F-2 | **No** — the ≥ 336 clean gapless runs requirement is unchanged | executor requires the literal string `OK` in stored evidence → **stop T6-23** and obtain a decision ID |
| **4** | `02B` Decision 2 parameter 5 and `CO-6-05` state **"Never both ON (FDS §28.5)"**, but FDS §28.5 (`03:1120`) and AC-39 state only that the *sequence* `canonicalRead → soak → canonicalWrite` is untouched — the literal phrase does not appear there. Read as a *permanent* prohibition, no compliant end state exists (reads would have to come from the legacy ledger while writes go canonical) | §11.4, §13.2 rule 7 | the finalized wording is **quoted verbatim** everywhere it is used; the plan reads the prohibition as governing **the window and the transition** — `canonicalWrite` never ON during the soak (F-4), never enabled before `E6-24`, never flipped in the same step as `canonicalRead`. Both flags end ON only in the post-cutover steady state (canonical read, canonical write, legacy read-only), which is what `reservation-pickup-cascade.service.ts:94-95` implements | **No** — nothing is relaxed: every flag action stays inside T6-21…T6-25 with its gate evidence | an executor reads the rule as permanently forbidding both ON → **stop before T6-25** and obtain a decision ID rather than choosing a read path that would strand the cutover |
| **5** *(not a conflict — sequencing consequence)* | `02B` makes `gba.reconciliation.enabled` a precondition of the soak while `TD-6-11` / `OI-28` remains `DEFERRED` as a *rule* | §8.4 Interpretation 2, §13.4 | T6-21 executes a sequencing consequence already named by `02B`; no activation rule is created; `OI-28` status unchanged (`DEFERRED`) | **No** | any proposal to generalise this into an activation order → decision ID required |

**No other reading exists in this plan.** Rows 1–3 are the three tier-F instances named by the §2.1 conflict rule; row 4 is an ambiguity internal to tier A; row 5 is not a conflict. Any further reading encountered at execution falls under §15.2.

---

## 18. Source Integrity Record

The four inputs below are **FROZEN**. This plan does not modify them, and no task in this plan modifies them (`L-r-25`, standing prohibitions 3–4). Recorded at authoring so that a later executor can prove nothing was rewritten.

| Input | Bytes | SHA-256 (first 16) | Status |
|---|---|---|---|
| `docs/availability/phase-6/01_FORENSIC_AUDIT.md` | 60,728 | `55FAF151CBDDBAFE` | FROZEN — unmodified |
| `docs/availability/phase-6/02_DECISION_RESOLUTION_ANALYSIS.md` | 95,522 | `DEC5B571693179E0` | FROZEN — unmodified |
| `docs/availability/phase-6/02A_OPEN_DECISION_EVIDENCE_RESOLUTION.md` | 59,549 | `C20B696EABD5B42F` | FROZEN — unmodified |
| `docs/availability/phase-6/02B_OWNER_DECISION_RESOLUTION.md` | 48,880 | `9034D63FADFE5682` | FROZEN — unmodified |

**Workspace baseline at authoring** (local tree is authority; no fetch/pull/restore/reset): `git status --porcelain` reports **787 entries — 368 modified tracked files, 170 untracked entries — against 1,494 tracked files.** Those counts are **identical before and after this document was written**: `docs/availability/phase-6/` is itself untracked, so it collapses to a single `??` entry and authoring this plan adds **no tracked change** (`git status --porcelain -uall` lists the four inputs and `03_IMPLEMENTATION_PLAN.md` individually, every one of them `??`). The pre-existing dirt is untouched: **authoring this plan changes no tracked file**; the only planned writes to tracked files are the two annotations in T6-07 and T6-08, each gated and performed under *annotate, never rewrite*.

**Not modified by this plan:** every file under `docs/availability/phase-4/`, `docs/availability/phase-5/`, `docs/enterprise/`, `docs/audit/` — annotation of a closed document is itself gated (T6-07, T6-08) and performed under *annotate, never rewrite*.

**Re-verification duty:** hashes and counts are re-checked at execution (T6-01, `E6-01`, `TD-6-12`) rather than trusted from this table.

---

## 19. Pre-Issue Internal Audit

Performed against this document as written, immediately before issue. **Method:** mechanical scans of the file plus manual confirmation that each quoted decision matches its source. **A `PASS` below records the completeness and internal consistency of this plan only. It is not execution evidence (`E6-nn`), satisfies no `02` §H gate, and authorises no work.**

| # | Check | Method | Result |
|---|---|---|---|
| **1** | Task register complete, contiguous, correctly namespaced | count and sequence `^#### T6-nn` | **PASS** — 28 tasks, `T6-01` … `T6-28`, no gaps, no duplicates |
| **2** | Every task carries a non-empty decision authority | per-block scan of `- **Decision authority:**` | **PASS** — 28/28 non-empty; each cites a `BD-6-*` / `TD-6-*` / `CO-6-*` / `DS-*` / `E-*` or a ratified Phase 3/4/5 rule |
| **3** | Every task's evidence entry matches its own number; register complete | per-block `- **Evidence:** E6-nn` vs task ID; §14.2 row count | **PASS** — 28/28 matched; §14.2 holds 28 rows, `E6-01` … `E6-28` |
| **4** | No foreign namespace used as a task, gate or evidence ID | heading and body scans | **PASS** — 0 `#### T5-*` headings, 0 `#### OI-*` headings, **no new gate namespace** (no identifier of the blocker/gate family is defined by this document; the `02` §H gates keep their existing IDs), 0 `S3R-*`; every `T5-nn` and `E-1…E-8` mention is a citation or a dependency only (standing prohibitions 5–6) |
| **5** | The seven `02A` §L.3 C4 items never become plan scope | context review of every occurrence | **PASS** — the six non-`OI-28` items occur only in §4.2 (the exclusion fence); `OI-28` occurs there, in this audit row, and as `DEFERRED` status citations (§7, §8.4, §11.1, §13.1, §13.4, §17). None of the seven is a task, a scope entry, or an action |
| **6** | Both ratified sequences intact; no stage invented | order comparison at every occurrence; scan for numbered "Stage n" vocabulary | **PASS** — both sequences appear only in the ratified order wherever they are written (standing prohibition 8, §3 invariants 11 and 18, §8.1, §15.1 item 6, §17), and the expanded flag chain in §11.1 preserves the same order; 0 invented stage names; §8.3 states its groupings are *"convenience groupings for a human executor, not ratified stages"* |
| **7** | `02B` Decision 1 (OI-03) reproduced without softening | RC-1…RC-5, date, fallback language | **PASS** — all five RC conditions tabled in §12.1 and repeated in T6-16; *"no calendar date"*, *"No fallback is pre-authorised"*, Options B/C recorded as rejected |
| **8** | `02B` Decision 2 (OI-04) reproduced in full | five parameters, SO-1…SO-4, F-1…F-5, restart/forward hold | **PASS** — all five parameters tabled verbatim at §11.2 (duration, threshold, evidence, failure, restart/rollback/forward hold), the failure table at §11.3, and the gate linkage at §11.1; no parameter softened, rounded or summarised away |
| **9** | `02B` Decision 3 (OI-07) Option A reproduced | additive field + calculator combination + FDS §11.3 annotation + three DB-backed tests + `/reconciliation` re-baseline | **PASS** — all five elements present in T6-09/T6-10 and summarised in §16.3 |
| **10** | `02B` Decision 4 (OI-10) followed exactly | two documents, `T6-nn` / `E6-nn`, no new gate namespace, no stages; `04_EXECUTION_EVIDENCE.md` **not** created | **PASS** — convention stated in the front matter and §14.1; `Test-Path docs/availability/phase-6/04_EXECUTION_EVIDENCE.md` → **False**; `03_IMPLEMENTATION_PLAN.md` is the only Phase 6 document created |
| **11** | `02` §I legacy dispositions preserved | §12.3 row count against `02` §I | **PASS** — 16 rows, `G-1` … `G-16`; `KEEP` / `DEFER` / `REMOVE` transcribed; no item dropped or reclassified |
| **12** | `02` §H cutover exit gates 1–9 all mapped to evidence | §16.1 row count | **PASS** — 9/9 gates, each with at least one `E6-nn` reference |
| **13** | No code, schema, DB, flag, test, reconciliation or cutover action performed or authorised outside a gated task | workspace inspection against standing prohibition 1 | **PASS** — the only file written is this one; no code edit, migration, test run, flag change, reconciliation run, cutover or legacy deletion occurred while authoring |
| **14** | Inputs unmodified; workspace baseline unchanged | MD5 of the four inputs and `git status` before/after | **PASS** — `8D7D8B6BC024` / `BC6B9D6327A7` / `FFB0118CF74E` / `0DD6C1E9931C` unchanged; 787 / 368 / 170 / 1,494 unchanged; no tracked file modified (§18) |
| **15** | Cross-references resolve | extract every `§n(.m)` reference, separate own-document targets from external citations (`FDS`, `02`, `phase-5/…`) by context, test against the heading index | **PASS** — **0 unresolved own-document references** (§1.2 … §18 all exist); unmatched references are external FDS / Phase-5 section citations, each verified in context |
| **16** | Dependency order is executable | unique `T6-nn` coverage of §8.2 and §8.3; entry gates per order row | **PASS** — both cover all 28 tasks; the graph is acyclic (WS-1 → WS-6 → WS-7 → WS-8, with WS-2 / WS-3 / WS-4 / WS-5 independent); every §8.3 order row names its entry gate |
| **17** | Document integrity and issue readiness | UTF-8 decode, BOM, encoding, heading inventory, front-matter promises | **PASS** — 19 numbered sections, 28 task headings, **0 replacement characters**, no BOM, UTF-8 punctuation intact; the front-matter promise *"Section 14 defines the `E6-nn` entries"* is satisfied |

**Not claimed by this audit:** re-verification of the four frozen source documents' internal consistency (they are authorities, not deliverables); correctness of anything at execution time; any `E6-nn` record. Findings from execution are recorded in `04_EXECUTION_EVIDENCE.md`, never here.

---

**Issued as a planning artifact only.** `03_IMPLEMENTATION_PLAN.md` records how already-finalized decisions would be executed. It creates no rule, closes no gate, and performs no work. Execution begins only when someone starts `T6-01` and opens `04_EXECUTION_EVIDENCE.md`.

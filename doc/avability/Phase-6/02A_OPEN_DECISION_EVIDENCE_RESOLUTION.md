# Phase 6 — 02A Open Decision & Evidence Resolution

**Artifact type:** resolution / closure artifact (Phase 6).
**Companions:** `01_FORENSIC_AUDIT.md` (forensic baseline), `02_DECISION_RESOLUTION_ANALYSIS.md` (decision baseline).
**Status of this document:** it resolves open items registered by the two companions. It creates no business rule, re-opens no Phase 5 status, and performs no implementation.

**Standing prohibitions (restated, binding on everything below):**

- No production code, schema, migration, DB data, test, defect fix, reconciliation run, cutover step, flag flip, or legacy deletion.
- No new business rules, no re-audit, no re-run of the full `02` analysis, no Implementation Plan, no implementation task list.
- No back-door implementation. Where this document writes "Implementation required: YES", it means *a plan must carry it* — not that it has been, or may be, done here. This applies explicitly to `TD-6-09`, `T5-59`, the matrix alias, A3 raw SQL, reconciliation machinery, population tooling, and legacy writer retirement.
- No rewriting of closed Phase 4 / Phase 5 documents (`annotate`, never rewrite — `TD-6-07`).
- No GitHub, no fetch/pull/restore. Phase 5 remains closed.

---

## A. Purpose

`02_DECISION_RESOLUTION_ANALYSIS.md` dispositioned the Phase 6 decision surface but deliberately did not close it: it ended with **8 rows in §K "Requiring Business Confirmation"**, **3 entry blockers still `OPEN` (X-1, X-4, X-7)** and **2 partially dispositioned (X-5, X-6)**, **12 open questions in §S**, **3 unresolved rows in §G.3**, **1 unresolved Phase 5 carry (`T5-11`)**, and a registered set of `EVIDENCE GAP` / `UNRESOLVED — REQUIRES EXPLICIT DECISION` statuses across `BD-6-*`, `TD-6-*`, and `CO-6-04`.

This document exists to do exactly one thing: **walk that open set, apply current local evidence, and state for each item whether it is now genuinely closed, still needs an owner, still needs evidence, blocks planning, gates only execution, is deferred, or is not applicable.**

It is *not* a Business Rules phase. It does not ratify a single business choice. Where evidence narrows a technical question, it decides it under authority (B) below; where a genuine business choice remains, it stops and records the question.

The output of this document is a single verdict in **§L**: is Phase 6 ready for Implementation Planning, ready subject to gates, or not ready.

---

## B. Authoritative Baseline

### B.1 What is authoritative

| Layer | Artifact | Role here |
|---|---|---|
| Ratified rules | `phase-5/03_FINAL_DOMAIN_SPECIFICATION.md` (`P-1…P-22`, `BR-5-001…BR-5-046`, `REQ-25.1…25.3`, `INV-*`, §13.3 input ownership, §24–§28) | **Highest.** Never redefined, never re-asked. |
| Ratified rules | `phase-4/13_FINAL_DOMAIN_SPECIFICATION.md`, `phase-3` plan + readiness review | Cross-phase authority where cited. |
| Phase 5 closure | `phase-5/06_EXECUTION_EVIDENCE.md:2088-2092` (12/12 gates) | **Closed.** Nothing here re-opens it. |
| Forensic baseline | `phase-6/01_FORENSIC_AUDIT.md` | Facts about the code as-built. |
| Decision baseline | `phase-6/02_DECISION_RESOLUTION_ANALYSIS.md` | Prior dispositions. Reviewed here only where §D permits. |
| Code as-built | `apps/**`, `packages/**`, `gateway/**` | Facts. Read-only inspection only. |

### B.2 Already terminal in `02` — **not re-reviewed here**

These carry a terminal status in `02` and are excluded from the inventory in §C: `BD-6-04` (decided by P-21; execution only), `BD-6-06` (`RESOLVED BY EXISTING DOMAIN RULE` → `DIRECT TECHNICAL CORRECTION REQUIRED` — decision closed, implementation enters plan scope), `TD-6-01` *fix half*, `TD-6-02` *disposition half*, `TD-6-03` *boundary + mechanism halves*, `TD-6-04`, `TD-6-05` *interim half*, `TD-6-07`, `TD-6-12`, `CO-6-01/02/03/05/06/07`, and the full `L-r-01…L-r-25`, `M-r-*`, `N-r-*`, `O-r-01…O-r-14` registers.

### B.3 Flag posture (as-is, carried from `02` J-AUD-01)

Only `FEATURE_GBA_A3_AUTHORITATIVE=true` (`.env:68`). The other six `gba.*` flags are absent from `.env` → false, and all seven read `false` in `.env.example:101-106`. Phase 6 must start from an **as-is** census, never from the documented end-state. No flag is changed by this document.

---

## C. Complete Open-Item Inventory

Every item `02` did **not** terminate. `OI-nn` are local identifiers (see `TD-6-15`); they carry no phase/task authority and are not a document-tree invention.

| ID | Source ref(s) in `02` | Subject | Previous status in `02` |
|---|---|---|---|
| OI-01 | BD-6-01, §K-1, G-5, S-6 | `room_inventory` — KEEP read-only or REMOVE | `REQUIRES OWNER DECISION` |
| OI-02 | BD-6-02, §K-2, G-8, X-4 | `rate_restrictions` steady state after E-1 | `UNRESOLVED — REQUIRES EXPLICIT DECISION` (contingent E-1) |
| OI-03 | BD-6-03, §K-3, G-3, X-1, S-1 | External `POST /rates/engine/modify` end-of-life date | `EVIDENCE GAP` (E-5) → `REQUIRES OWNER DECISION` |
| OI-04 | BD-6-05, §K-4, S-4 | `canonicalRead` → soak → `canonicalWrite` soak exit criteria | `REQUIRES OWNER DECISION` |
| OI-05 | BD-6-07, §K-5, G-14, S-5 | `T5-36` GBA `pickupPct` client math vs sanctioned allowlist | `REQUIRES OWNER DECISION` |
| OI-06 | BD-6-08, §K-6, X-5, S-10 | BLK-1 / BLK-2 wash deviations (build or remove wash) | `DEFERRED` + `REQUIRES OWNER DECISION` |
| OI-07 | TD-6-09, §K-7, S-3, CO-6-04, D-AUD-01 | Canonical read omits assertion-balance consumption | `RECOMMENDED — CONFIRMATION REQUIRED` |
| OI-08 | TD-6-10, §K-8 | Raw SQL identifier/filter interpolation on A3 surface | `RECOMMENDED — CONFIRMATION REQUIRED` |
| OI-09 | TD-6-14, S-8, I-6 | Assertion-lifecycle outbox event vs scheduled poll | `UNRESOLVED — REQUIRES EXPLICIT DECISION` |
| OI-10 | TD-6-15, X-7, S-9 | Phase 6 document set, stage model, task-ID space | `UNRESOLVED — REQUIRES EXPLICIT DECISION` |
| OI-11 | TD-6-08, S-7, §J `T5-11` | Interim-gate honesty test — re-derive or subsume | `UNRESOLVED — REQUIRES EXPLICIT DECISION` |
| OI-12 | TD-6-01 (parameterisation half) | Parameterise A3 raw-SQL filters now or alias-fix only | `RECOMMENDED — CONFIRMATION REQUIRED` |
| OI-13 | TD-6-02 (timing half), H-1 | `crs.modifyReservation` retire/port **timing** | `EVIDENCE GAP` (E-5) |
| OI-14 | TD-6-03 (cadence), §G.3 row 7 | Reconciler cadence | `RECOMMENDED — CONFIRMATION REQUIRED` |
| OI-15 | TD-6-03 (persistence), §G.3 row 5 | Where reconciliation findings persist | `RECOMMENDED — CONFIRMATION REQUIRED` |
| OI-16 | TD-6-05 (long-term), H-3, X-4 | `evaluateRestrictions` — authority consult vs `rate_restrictions` read | `EVIDENCE GAP` (E-1) |
| OI-17 | CO-6-04, §G.3 row 6 | What baseline does cutover measure against? | `UNRESOLVED — REQUIRES EXPLICIT DECISION` |
| OI-18 | E-1 (population half) | Per-hotel row counts + recency for the six store families | `OPEN — hard` |
| OI-19 | E-1 (writer half) | Writer identification for the six store families | `OPEN — hard` |
| OI-20 | E-3 | Runtime confirmation of the production DI chain | `OPEN — hard` |
| OI-21 | E-4, S-11 | Meaning of `restrictions (rate_code='CUTOFF')` and `zero_sell_value` | `OPEN — non-blocking` |
| OI-22 | E-5, S-1, X-1 | Inventory of out-of-repo consumers of `/rates/engine/*` | `OPEN — non-blocking for disposition` |
| OI-23 | E-6 | Live behavior of OTA overbooking branch | `OPEN — informative` |
| OI-24 | E-7, S-12 | Runtime winner of duplicate `GET /tax-rates` | `OPEN — non-blocking` |
| OI-25 | E-8 | Reconciled executed baselines vs §A6 baselines | `OPEN — first executed exit` |
| OI-26 | E-2 | Precedence intent if store conflicts exist | `DEFERRED (conditional)` |
| OI-27 | TD-6-06, N-5 | Duplicate `GET /tax-rates` route merge | `DEFERRED` |
| OI-28 | TD-6-11, X-6 (J-4/J-5) | Activation order of `gba.reconciliation.enabled` / `twoLayerConsult` | `DEFERRED` |
| OI-29 | TD-6-13, G-14/G-15/G-16 | Allowlist / dead-DI removal timing | `DEFERRED` |
| OI-30 | BD-6-09, BD-6-10 | Analytics contract / channel publication | `DEFERRED` (P-19, P-20) |

**Total open items reviewed: 30.**

Every `§S` question S-1…S-12 maps to a row above (S-1→OI-22, S-2→OI-18+OI-19, S-3→OI-07, S-4→OI-04, S-5→OI-05, S-6→OI-01, S-7→OI-11, S-8→OI-09, S-9→OI-10, S-10→OI-06, S-11→OI-21, S-12→OI-24). Every `§F` blocker maps: X-1→OI-22+OI-03, X-4→OI-02+OI-16, X-5→OI-06, X-6→OI-04+OI-28, X-7→OI-10.

---

## D. Resolution Method

### D.1 Evidence quality labels

`PROVEN` — directly readable in this repository, cited to `file:line`.
`SUPPORTED` — strongly implied by repository evidence plus a ratified rule, but not literally stated.
`UNKNOWN` — not obtainable from this repository; never guessed, never substituted by assumption.

### D.2 Decision authority (unchanged from `02` §0)

| Class | Authority | Treatment |
|---|---|---|
| (A) Facts | repo / DB / docs | Stated with `file:line`. |
| (B) Technical decisions resolvable from evidence | evidence in this repository | Decided here as `RESOLVED BY TECHNICAL EVIDENCE`. |
| (C) Existing ratified rules | Phase 4/5 specifications | Preserved verbatim; **never re-asked**. |
| (D) New business rules / policy choices | owner only | `OWNER DECISION REQUIRED`; options + consequences + recommendation only. Ratification prohibited. |

**Consequence for the owner queue.** An item is escalated to (D) only if it is a genuine business/domain/policy choice. Anything in class (B) — including scope-of-a-code-change and sequencing-of-a-technical-task questions — is decided here even if `02` had escalated it. `02`'s §K list is therefore re-tested item by item rather than inherited.

### D.3 The planning-blocker test (applied literally)

An item blocks planning **only if** the Implementation Plan cannot safely define **scope, dependency, behavior, verification, or execution order** without knowing the answer.

- If the plan can carry it as a **conditional / gated task** with both branches defined → **BLOCKS NEITHER**.
- If the plan content is fully definable but execution must wait for a precondition → **BLOCKS EXECUTION ONLY**.
- If the plan document itself cannot be safely authored → **BLOCKS PLANNING**.

### D.4 Rules of closure

1. Do not re-ask what a ratified rule already decides (D.2 class C).
2. Do not accept `02`'s escalation at face value — re-test against D.2/D.3.
3. An evidence gap that a **registered plan task** is designed to close (census, runtime observation) is not a permanent gap: it is `EVIDENCE REQUIRED` with a named deliverable, and its planning impact is assessed separately.
4. A gap that the repository can close **today** is closed today, with the exact evidence recorded — and with an explicit statement of **what that evidence does not prove**.
5. Where current evidence **contradicts or materially refines** a `02` statement, the contradiction is recorded explicitly with both sides (this occurs exactly once — see OI-17).

---

## E. Owner Decisions Required (complete list)

### E.1 Required *for Phase 6* — 4 items

#### OI-03 — External `POST /rates/engine/modify` end-of-life date

- **Source:** `BD-6-03`, `02` §K-3, blockers G-3 / X-1, question S-1.
- **Previous status:** `EVIDENCE GAP` (E-5) → `REQUIRES OWNER DECISION`.
- **Exact question:** On what date does the external `POST /rates/engine/modify` route stop serving?
- **Evidence reviewed:** `01` N-2/H-1; `phase-5/04_IMPLEMENTATION_PLAN.md:261,272` (T5-88, E-5 disposition); `phase-5/06_EXECUTION_EVIDENCE.md:100,2107`; `gateway/ingress/nginx.conf`; `gateway/api-gateway/kong.yml`; FDS §24.4 (`03:981`), BR-5-018/025.
- **Evidence result:** `PROVEN` that the **disposition is already fixed** — retirement is required by BR-5-018/025 and FDS §24.4, and T5-88 records E-5 as *"inventory unknown ⇒ in-repo retirements proceed, external-facing retirement waits (**disposition unchanged**)"*. `UNKNOWN` whether any external caller exists (no consumer registry anywhere in `gateway/` — see OI-22). A date cannot be inferred from any of this.
- **Resolution:** The *decision* (retire) is class (C) and closed. The *date* is class (D): an operational/commercial commitment that no rule fixes. **Genuinely requires the owner.**
- **Authority:** Product owner.
- **Planning impact:** **BLOCKS NEITHER.** Plan defines scope ("retire external `engine/modify`"), dependency (`E-5 → BD-6-03 → retirement`), behavior (route 404, no in-repo caller), verification (route returns 404 + writer scan), order (after OI-18 census) — all without a date. The date is a task attribute.
- **Execution impact:** `EXECUTION GATE` — no retirement may run until E-5 is returned **or** the owner accepts the residual risk of retiring blind, and a date is set.
- **Dependencies:** OI-22 (E-5). Recommendation `A` (retire now) only if E-5 comes back empty.
- **Required follow-up:** owner sets date; record in plan as a gated task.
- **Final status:** `OWNER DECISION REQUIRED`.

#### OI-04 — Soak exit criteria for `canonicalRead` → soak → `canonicalWrite`

- **Source:** `BD-6-05`, `02` §K-4, question S-4, blocker X-6.
- **Previous status:** `REQUIRES OWNER DECISION` (order = closed; criteria = open).
- **Exact question:** What does "soak" mean — duration, and evidence threshold, for the window between `gba.pickup.canonicalRead` ON and `gba.pickup.canonicalWrite` ON?
- **Evidence reviewed:** FDS §28.3 (`03:1105-1110`), §28.5 (`03:1120`), AC-39 (`03:1225`); `phase-5/04_IMPLEMENTATION_PLAN.md:163`; `phase-5/04_IMPLEMENTATION_PLAN.md:405-413` (T5-10); `.env.example:101-106`; `gba-reconciliation.service.ts:131`.
- **Evidence result:** `PROVEN` that the **sequence** is ratified and preserved verbatim in four places (`canonicalRead` → soak → `canonicalWrite`). `PROVEN` that **no document anywhere defines the soak's duration or threshold** — every citation preserves the word "soak" without a definition. The only *defined* soak in the corpus is **T5-10 "parity soak"** (`04:405-413`), which is a different thing entirely: flag-ON *value parity* between matrix and snapshot for the `gba.a3.authoritative` gate, not a pickup-ledger soak. Conflating the two would be an invention.
- **Resolution:** Order is class (C) → closed. Criteria are class (D) — an operational risk-tolerance choice with no derivable answer. **Genuinely requires the owner.**
- **Authority:** Product / ops owner.
- **Planning impact:** **BLOCKS NEITHER.** The plan defines the sequence task, its preconditions (`canonicalRead` ON), its gate (`BD-6-05` criteria recorded), and its successor (`canonicalWrite` ON). Criteria are a gate value.
- **Execution impact:** `EXECUTION GATE` — `gba.pickup.canonicalWrite` may not be turned on until criteria exist and are met. Never both at once.
- **Dependencies:** none for planning. If the owner adopts the parity-threshold option, OI-14 (hourly cadence, `gba.reconciliation.enabled`) becomes a *precondition* of the soak — a sequencing observation, not a new rule.
- **Required follow-up:** owner writes duration + threshold; plan records it as the J-7 gate.
- **Final status:** `OWNER DECISION REQUIRED`.

#### OI-07 — Should the canonical read include assertion-balance consumption?

- **Source:** `TD-6-09`, `02` §K-7, question S-3, new finding `D-AUD-01`, and (via OI-17) `CO-6-04`.
- **Previous status:** `RECOMMENDED — CONFIRMATION REQUIRED`.
- **Exact question:** Is `reservationConsumption` in the canonical read supposed to cover **all** reservation commitment, or only the legacy population? Options as registered: **A** correct additively / **B** declare a post-assertion contract / **C** document and defer.
- **Evidence reviewed:**
  - Read path: `snapshot-calculator.ts:14` (`consumption = reservationConsumption + gbaRemaining + allotmentRemaining` — three terms), `availability-snapshot.service.ts:88,98`; no reader of `availability_assertion_balances` anywhere in the read path.
  - Write path: `availability-assertion.service.ts:271-275` (first-touch assigns on assert), `:894-900` (`remaining = day.sellableAvailable − balances.get(stayDate)`).
  - Phase 3 authority: `availability-phase3-implementation-plan.md:613` (**G5** "No dual count — each Reservation counted by exactly one of {legacy source, assertion balances}"), `:829` ("ASSERTION_MANAGED Reservations are represented solely by assertion balances"), `:136` (X-13), `:398`; `availability-phase3-readiness-review.md:194` (`[VERIFIED FACT]`).
  - Phase 4 formula: `phase-4/02_CURRENT_ARCHITECTURE.md:131`, `phase-4/13_FINAL_DOMAIN_SPECIFICATION.md:448`.
  - Ratified rules: P-11 four fact kinds (`03:105,389,394` — fourth kind = "reservation commitment"), P-12 single source of the sellable number (`03:106`), FDS §11.3 payload (`03:477`), FDS §26.2 balances = domain state (`03:1032`), FDS `:691` (assertion balances have their **own** sanctioned read — `gba.pickup.twoLayerConsult`).
  - Visibility: matrix authority branch sets `available: fact.sellableAvailable` (`availability-sales.controller.ts:494`, `:539`); page renders `cell.available` (`AvailabilityPage.tsx:1053`); hook consumed at `AvailabilityPage.tsx:27,788`.
  - Non-evidence: T5-10 parity soak compares matrix against snapshot only and seeds no ASSERTION_MANAGED rows.
- **Evidence result:**
  - `PROVEN` — the read omits balances; the three-term formula is the ratified combination; balances have a separate sanctioned read path; **G5 is satisfied** (no double count — a global no-dual-count guard is not an at-least-once guard).
  - `PROVEN` — the omission is **operator-visible** (matrix `available` cell, flag currently ON).
  - `UNKNOWN` — whether the *intended* read contract is balance-inclusive. `02`'s own finding stands verbatim: *"nothing in the corpus declares which is intended."* Neither P-11/P-12 (argue for inclusion) nor the Phase 4 three-term formula + the separate `twoLayerConsult` read (argue for exclusion) is dispositive, because the formula predates assertion balances.
- **Resolution:** **Option B is ruled out by class (C)** — it requires every reader to subtract balances itself, which `02` correctly shows violates P-11/P-12 and is impossible for frontends. **Options A and C both remain consistent with every ratified rule**, so this is a real choice between "amend the published read contract now" and "document the gap and defer". It touches a Phase 5–published contract (`FDS §11.3`) and changes an operator-visible number → class (D).
- **Authority:** Engineering owner for the technical change; **product owner acknowledgement** required because `sellableAvailable` is operator-visible.
- **Planning impact:** **BLOCKS NEITHER.** The plan carries a conditional task with both branches defined: *if A* — additive field `assertionBalanceConsumption`, legacy-population field byte-identical, DB-backed test per surface (snapshot, matrix, reconciliation), dated `annotate` note against FDS §11.3; *if C* — documented-gap note plus an explicit statement that capacity baselines are withheld. Verification and behavior are fully specified either way.
- **Execution impact:** `EXECUTION GATE` for **capacity-grade** evidence only — see OI-17 for the refined sequencing rule.
- **Dependencies:** none. Must precede any absolute-capacity baseline.
- **Required follow-up:** owner sign-off (A or C); if A, a dated amendment note against FDS §11.3 (annotate, never rewrite). Downstream notice owed to Reservations Phase 9 (`02` §R).
- **Final status:** `OWNER DECISION REQUIRED`.

#### OI-10 — Phase 6 document set, stage model, task-ID space

- **Source:** `TD-6-15`, blocker X-7, question S-9.
- **Previous status:** `UNRESOLVED — REQUIRES EXPLICIT DECISION`, deliberately not decided in `02`.
- **Exact question:** What is Phase 6's document set, stage model, and task-ID space?
- **Evidence reviewed:** `01` §C.4 D-4/D-5/D-6/D-7; `phase-5/04_IMPLEMENTATION_PLAN.md` §33 (Phase 5's own deferral register); the two Phase 6 artifacts that exist.
- **Evidence result:** `PROVEN` that no rule in either precedent yields a unique Phase 6 list, and `01` D-6 forbids inheriting `T5-nn` / `BLK-P5-nn` / `E-n` / `S3R-nnn`. Asserting an answer would be invention — which both `02` and this document are prohibited from doing.
- **Resolution:** Class (D), but a **process** class: a phase-owner call, not a business rule. It is nonetheless the one item whose answer is needed **before a plan document is written**, because a plan cannot be authored without an ID convention and section structure.
- **Authority:** Phase owner.
- **Planning impact:** **BLOCKS PLAN AUTHORING ONLY.** It does **not** block the definition of scope, dependency, behavior, verification, or execution order — those are fully specified in `02` §R regardless of the ID prefix. It is nevertheless a precondition to *drafting*.
- **Execution impact:** none.
- **Dependencies:** none.
- **Required follow-up:** one-line phase-owner call (document list + stage headings + ID prefix). Must be recorded before the Implementation Plan is authored.
- **Final status:** `OWNER DECISION REQUIRED`.

### E.2 Owned but **not Phase 6's** to decide — 2 items

These require an owner, but they are outside Phase 6's scope (`02` §R does not enumerate them in plan scope) and do not gate Phase 6.

#### OI-05 — `T5-36` GBA `pickupPct` client math vs sanctioned allowlist

- **Source:** `BD-6-07`, `02` §K-5, G-14, S-5.
- **Previous status:** `REQUIRES OWNER DECISION` (scope).
- **Exact question:** Complete `T5-36` (remove client math), or permanently sanction `CLIENT_MATH_ALLOWED` for `pickupPct`?
- **Evidence reviewed:** `phase-5/04_IMPLEMENTATION_PLAN.md:832` (`T5-36` task), `:1108` (AC-20), `:1143` (INV-18); `phase-5/06_EXECUTION_EVIDENCE.md` (task never appears — L-3); live sites `GroupBookingsListView.tsx:101`, `GroupBookingDetailView.tsx:101,109,114`.
- **Evidence result:** `PROVEN` `T5-36` is unevidenced and the math is live. `PROVEN` the question is genuinely ambiguous: `02` correctly observes BR-5-012/AC-20 govern *availability* arithmetic (sellable/available) while `pickupPct = picked ÷ contracted` is a GBA block metric. Both readings are legitimate.
- **Resolution:** A real class (D) scope choice — **but it is a GBA/frontend display decision with no bearing on reconciliation, cutover, or legacy retirement**, which is what Phase 6 is. `02` §R itself excludes it from the enumerated plan scope.
- **Authority:** Product owner, via the GBA / Reservations-frontend workstream.
- **Planning impact:** **BLOCKS NEITHER** (Phase 6).
- **Execution impact:** none for Phase 6.
- **Dependencies:** none.
- **Required follow-up:** decision owed by its owning workstream; `G-14` allowlist entries retire as sites do.
- **Final status:** `DEFERRED` — *deferred out of Phase 6 scope; the decision remains owed elsewhere.*

#### OI-06 — BLK-1 / BLK-2 wash deviations

- **Source:** `BD-6-08`, `02` §K-6, blocker X-5, question S-10.
- **Previous status:** `DEFERRED` + `REQUIRES OWNER DECISION`.
- **Exact question:** Implement `GUARANTEED_BLOCK` wash exclusion (BLK-1 / Deviation A) and a durable wash/release/attrition store (BLK-2 / Deviation B), or remove the wash feature?
- **Evidence reviewed:** `phase-5/04_IMPLEMENTATION_PLAN.md:1410-1425` (Phase 5 carry-over table — *open; wash flag stays blocked; no Phase 5 task activates wash*), §33; FDS §28.1/§28.5 wash row; `02` X-5.
- **Evidence result:** `PROVEN` these are Phase 4 carry-overs that Phase 5 was forbidden to execute and did not; `PROVEN` their **only** effect is gating `gba.wash.schedulerEnabled`, which is already false and outside Phase 6 scope.
- **Resolution:** Deferral is correct and ratified (`02` X-5, Phase 5 plan §32). The build-or-remove choice is class (D) but belongs to whoever owns wash activation.
- **Authority:** Product owner (Phase 4 carry-over ownership).
- **Planning impact:** **BLOCKS NEITHER.**
- **Execution impact:** none for Phase 6; wash may not be activated under any Phase 6 heading.
- **Dependencies:** none.
- **Required follow-up:** decision owed by the GBA/wash workstream.
- **Final status:** `DEFERRED` — *deferral confirmed; decision remains owed elsewhere.*

---

## F. Evidence Gap Resolution (complete list)

### F.1 E-1 — the six store families: **split, half closed today**

#### OI-19 — Writer identification  → **CLOSED**

- **Source:** E-1 (writer half), `02` X-4, question S-2 (writer component).
- **Previous status:** `OPEN — hard`.
- **Exact question:** Who writes `room_inventory`, `rate_restrictions`, `out_of_order`, `out_of_service`, and the six A3 restriction tables?
- **Evidence reviewed (this document, exhaustive repo scan excluding tests):**

| Store family | In-repo writer | Evidence |
|---|---|---|
| `room_inventory` | **NONE** | Readers only: `availability-sales.controller.ts:86` (`COALESCE(ri.available_rooms, …)`), `:99` fallback, `:116`; `packages/db/schema.prisma:10413`; `add_housekeeping_tables` migration SQL. Zero `create`/`update`/`upsert` on the model anywhere. |
| `rate_restrictions` | **NONE** | Exactly four non-test references, all reads or comments: `unresolved-restriction.adapter.ts`, `prisma-restriction.adapter.ts:114,153` (marks store `unproven`), `restriction-write.contracts.ts:7` (comment), `crs-engine.service.ts:146-196` (hard-block read). |
| `out_of_order` | **NONE** | `availability-source.adapter.ts:75` (`findMany`), `availability-sales.controller.ts:401` (raw `LEFT JOIN`). |
| `out_of_service` | **NONE** | `availability-source.adapter.ts:76` (`findMany`), `availability-sales.controller.ts:403` (raw `LEFT JOIN`). |
| six A3 restriction tables (`close_to_arrival`, `close_to_departure`, `zero_sell_limits`, `room_type_sell_limits`, `minimum_length_of_stay`, `maximum_length_of_stay`) | **IDENTIFIED** — `PrismaRestrictionWriteRepository` (T5-45) via `RestrictionWriteService` ← A3 write endpoint in `availability-sales.controller.ts`, registered in `availability.module.ts` | `restriction-write.repository.ts:19-146` (Prisma `upsert` only, zero SQL text, every predicate carries `hotel_id`). |

- **Evidence result:** `PROVEN` for all five families: **no in-repo writer exists for `room_inventory`, `rate_restrictions`, `out_of_order`, or `out_of_service`; the six A3 tables' writer is the authority-side restriction write path (DS-04), already identified.**
- **What this does NOT prove:** it does **not** prove no writer exists anywhere. An external system, a migration, or direct SQL could write these tables. Recency and per-hotel population are exactly what would distinguish "live external writer" from "dormant table" — and those are database facts, not repository facts (OI-18).
- **Resolution:** Class (A) facts, closed today under D.4 rule 4.
- **Authority:** repository evidence.
- **Planning impact:** **BLOCKS NEITHER.**
- **Execution impact:** none directly; it sharpens OI-02 and OI-16 (below).
- **Dependencies:** none.
- **Required follow-up:** carry the negative result into OI-18's census as the writer column.
- **Final status:** `CLOSED`.

#### OI-18 — Per-hotel row counts + recency  → **EVIDENCE REQUIRED**

- **Source:** E-1 (population half), `02` X-4, question S-2 (population component).
- **Previous status:** `OPEN — hard`.
- **Exact question:** For each hotel, how many rows exist in each of the six store families, and when was each last written?
- **Evidence reviewed:** FDS §31 E-1 (`03:1241`); `phase-5/04:258` (E-1 acceptance: *"store + count/recency/writer-or-UNKNOWN per hotel … **no default assumptions**"*); BR-5-046 (`03:1324`); `02` CO-6-01.
- **Evidence result:** `UNKNOWN` — row counts and write timestamps are database state, not repository state, and are not present in any document. **This document does not invent them and does not run them.**
- **Resolution:** `EVIDENCE REQUIRED`. Crucially, this gap has a **registered deliverable**: the read-only census is the first task of `TD-6-04` / `CO-6-02` (`02` recommends *"B preceded by a read-only census … only a census establishes the denominator"*). It is a plan task, not an unknown.
- **Authority:** database read-only census.
- **Planning impact:** **BLOCKS NEITHER** — the census *is* plan scope.
- **Execution impact:** **HARD GATE.** CO-6-01 (ratified): no legacy writer may be removed before E-1 closes. DS-02 execution (frontend cutover to authority numbers) requires DS-01 operational, which requires E-1 (`03:957-971`).
- **Dependencies:** OI-19 (writer column now known).
- **Required follow-up:** read-only census; store + count + recency + writer-or-UNKNOWN per hotel.
- **Final status:** `EVIDENCE REQUIRED`.

### F.2 The remaining registered evidence gaps

All of these are Phase 5–registered (`FDS §31`, `03:1235-1250`; `phase-5/04:258-274`) and are re-listed here with current planning/execution assessment. None is re-opened; none is closed by this document.

#### OI-22 — E-5: out-of-repo consumers of `/rates/engine/*`

- **Previous status:** `OPEN — non-blocking for disposition` (blocker X-1, question S-1).
- **Evidence reviewed (this document):** `gateway/ingress/nginx.conf` — generic reverse proxy, `location /api/` → `api_upstream`, **no per-route consumer list**; `gateway/api-gateway/kong.yml` — a single catch-all `/api/v1` route with `jwt`/`rate-limit`/`cors` plugins plus a webhook `ip-restriction`, **`routes/` and `plugins/` directories empty, no consumer registry**.
- **Evidence result:** `PROVEN` that **no in-repo consumer registry or per-route consumer inventory exists**. `UNKNOWN` whether external callers exist — `02`'s characterization is correct and stands: *"Cannot be answered in-repo."*
- **What this does NOT prove:** it does not prove the route is unused. Absence of a registry is not absence of traffic.
- **Resolution:** `EVIDENCE REQUIRED` (external: gateway/access logs or an ops query — outside the repository). **Planning impact is settled by a ratified rule, not by opinion:** `phase-5/04:261,272` records E-5 as *"OPEN — non-blocking for disposition"* with the disposition *"inventory unknown ⇒ in-repo retirements proceed, external-facing retirement waits (**disposition unchanged**)"*. That rule governs Phase 6 identically.
- **Authority:** ops/gateway inventory.
- **Planning impact:** **BLOCKS NEITHER.** Dispositions of `TD-6-02` and `BD-6-03` are already fixed by BR-5-017/§21.4.
- **Execution impact:** `EXECUTION GATE` — gates the `BD-6-03` date and the `TD-6-02` retirement timing only.
- **Required follow-up:** obtain inventory outside the repository.
- **Final status:** `EVIDENCE REQUIRED` (planning impact: none).

#### OI-20 — E-3: runtime confirmation of the production DI chain

- **Previous status:** `OPEN — hard`.
- **Evidence reviewed:** FDS §28.3 ON-gate condition 3 (`03:1108`); BR-5-037; BR-5-009 (stub bound ⇒ flag must remain OFF); `.env:68` (flag **is** ON — `02` J-AUD-01 posture).
- **Evidence result:** `UNKNOWN` — requires executed observation in a safe environment, not a repository read.
- **Resolution:** `EVIDENCE REQUIRED`. Note the posture tension this closes: the flag is ON while FDS §28.3's four-condition gate is not fully evidenced (evaluator now bound per T5-09; E-1 population and E-3 runtime outstanding). `02` J-AUD-01 stands — Phase 6 must record the as-is census rather than assume the documented end-state.
- **Planning impact:** **BLOCKS NEITHER.**
- **Execution impact:** `EXECUTION GATE` — precondition of BR-5-037 closure and of any forward flag action.
- **Required follow-up:** recorded request/response/rollback evidence.
- **Final status:** `EVIDENCE REQUIRED`.

#### OI-21 — E-4: meaning of `restrictions (rate_code='CUTOFF')` and `zero_sell_value`

- **Source:** question S-11.
- **Previous status:** `OPEN — non-blocking`.
- **Evidence reviewed:** FDS §31 E-4; BR-5-024; `phase-5/04:274` (T5-87) — *"Failure: meaning unknown ⇒ store remains excluded forever in this phase (AC-31), no migration, evaluator scope unchanged."*
- **Evidence result:** `UNKNOWN` — product/domain definition, not in the repository.
- **Resolution:** `EVIDENCE REQUIRED`, and the failure mode is already specified: unknown ⇒ excluded forever this phase. The outcome is safe either way.
- **Planning impact:** **BLOCKS NEITHER.**
- **Execution impact:** none for Phase 6 — exclusion is the standing state.
- **Final status:** `EVIDENCE REQUIRED`.

#### OI-23 — E-6: live behavior of the OTA overbooking branch

- **Previous status:** `OPEN — informative`.
- **Evidence reviewed:** FDS §31 E-6; `phase-5/04:276` (T5-89) — *"none (rule already issued)"*.
- **Evidence result:** `UNKNOWN` (runtime observation). The business rule BR-5-020 stands regardless of the observation.
- **Resolution:** `EVIDENCE REQUIRED` but **informative only**.
- **Planning impact:** **BLOCKS NEITHER.**
- **Execution impact:** none — no Phase 6 task depends on it.
- **Final status:** `EVIDENCE REQUIRED`.

#### OI-24 — E-7: runtime winner of duplicate `GET /tax-rates`

- **Source:** question S-12.
- **Previous status:** `OPEN — non-blocking`.
- **Evidence reviewed:** FDS §31 E-7; `phase-5/04:278` (T5-90) — *"owner not changed until observed; no aliasing attempted."*
- **Evidence result:** `UNKNOWN` (runtime request trace).
- **Resolution:** `EVIDENCE REQUIRED`, and it gates only OI-27 (the merge itself), which is `DEFERRED` as non-availability hygiene.
- **Planning impact:** **BLOCKS NEITHER.**
- **Execution impact:** gates `TD-6-06` only, which is deferred.
- **Final status:** `EVIDENCE REQUIRED`.

#### OI-25 — E-8: reconciled executed baselines

- **Previous status:** `OPEN — first executed exit`.
- **Evidence reviewed:** FDS §31 E-8; `phase-5/04:280` — *"exit blocked until reconciled; baselines never silently redefined."*
- **Evidence result:** `UNKNOWN` — requires full test runs with a DB environment.
- **Resolution:** `EVIDENCE REQUIRED`. This is by construction an **executed-exit** item: it can only arise *after* code exists to run. It is the archetypal BLOCKS-EXECUTION-ONLY case.
- **Planning impact:** **BLOCKS NEITHER.**
- **Execution impact:** blocks every code-changing exit.
- **Final status:** `EXECUTION GATE ONLY`.

#### OI-26 — E-2: precedence intent if store conflicts exist

- **Previous status:** `DEFERRED (conditional)`.
- **Evidence reviewed:** FDS §31 E-2; BR-5-003; BR-5-046.
- **Evidence result:** not applicable unless OI-18's census surfaces an actual conflict. Standing behavior: conflict ⇒ `UNRESOLVED`, no invented precedence.
- **Resolution:** conditional; it does not exist as an item until evidence creates it.
- **Planning impact:** **BLOCKS NEITHER.**
- **Execution impact:** none unless triggered.
- **Final status:** `NOT APPLICABLE` *(conditional — activates only if conflicts are evidenced).*

---

## G. Technical Confirmation Items

### G.1 Items `02` escalated that this document resolves under authority (B)

Class (B) items are technical, have no business consequence, and are decidable from repository evidence. Three of `02`'s §K escalations fail that test and are closed here.

#### OI-08 — Raw SQL identifier/filter interpolation  → **CLOSED**

- **Source:** `TD-6-10`, `02` §K-8.
- **Previous status:** `RECOMMENDED — CONFIRMATION REQUIRED`.
- **Exact question:** Parameterise the A3 raw-SQL filters now, or alias-fix only?
- **Evidence reviewed:** `availability-sales.controller.ts:305-307` (`rtFilter`/`rtFilterRt`/`rcFilter` built by interpolation into `$queryRawUnsafe` with `replace(/'/g,"''")` quote-escaping only), `:377`, `:390` (roomType embedded directly).
- **Evidence result:** `PROVEN` — identifier interpolation is the mechanism producing the N-1/alias defect (`02` N-1/N-6 and the audit's own assessment). `PROVEN` — both questions live in the same code region as `TD-6-01`.
- **Resolution:** Class (B). Parameterisation has **zero** business consequence, is evidenced as the correct fix for the same defect `TD-6-01` must fix anyway, and splitting the change would double the regression surface over one region. Decided: **parameterise, bundled with `TD-6-01`.** `02`'s escalation of this to §K was over-cautious — it is engineering scope, not a product choice.
- **Authority:** engineering (technical evidence).
- **Planning impact:** **BLOCKS NEITHER.** Merge with the `TD-6-01` task in plan scope.
- **Execution impact:** same release as `TD-6-01`; DB-backed test required (existing mocked matrix specs catch neither).
- **Dependencies:** `TD-6-01`.
- **Required follow-up:** plan carries one task, not two.
- **Final status:** `CLOSED`.

#### OI-09 — Assertion-lifecycle outbox event vs scheduled poll  → **CLOSED**

- **Source:** `TD-6-14`, question S-8, inventory I-6.
- **Previous status:** `UNRESOLVED — REQUIRES EXPLICIT DECISION` (engineering owner).
- **Exact question:** Does Phase 6 reconcile by scheduled poll (`TD-6-03` option A) or by an assertion-lifecycle event (option B)?
- **Evidence reviewed:** `TD-6-03`'s already-taken decisions in `02` — boundary `RESOLVED BY EXISTING DOMAIN RULE` (REQ-25.x), mechanism `RESOLVED BY TECHNICAL EVIDENCE` = **scheduled report-only job**; Phase 3 plan `:278` (**"No outbox, saga or event bus may substitute"** for the Availability↔Reservation atomic operation) and `:287` (integration events may use the outbox only for matters *"unrelated to inventory"*); Phase 3 M-10 records the event as **MISSING**; `phase-5/04:85` (`@Cron` exists only in outbox plumbing); `gba-reconciliation.service.ts:131` (an hourly `@Cron` already exists as precedent).
- **Evidence result:** `PROVEN` the event is absent and **not required by any ratified rule**; `PROVEN` a scheduled job satisfies REQ-25.1 (sanctioned comparison), REQ-25.2 (report/never repair), and REQ-25.3 (evidence, not authority) with **zero new artifacts**; `PROVEN` building the event would require new publisher infrastructure that Phase 3's own rules restrict for inventory matters.
- **Resolution:** Class (B). Option **A (scheduled report-only job)** is the only option that satisfies every ratified requirement without introducing an unratified artifact. Option B is not forbidden — it is simply *unnecessary*, and unnecessary new infrastructure is a technical judgement, not a policy one. `02`'s own recommendation ("resolve it inside `TD-6-03` rather than as a standalone build") points the same way.
- **Authority:** engineering (technical evidence).
- **Planning impact:** **BLOCKS NEITHER.** `TD-6-03` proceeds as a scheduled job; I-6 is recorded as *not built, not required*.
- **Execution impact:** none.
- **Dependencies:** `TD-6-03`.
- **Required follow-up:** if an event is ever wanted, it must arrive as a new requirement with its own decision ID — it is not implied by Phase 6.
- **Final status:** `CLOSED` *(decision: scheduled poll; event not required).*

#### OI-12 — Parameterise now vs alias-fix only  → **CLOSED**

- **Source:** `TD-6-01` (parameterisation half).
- **Previous status:** `RECOMMENDED — CONFIRMATION REQUIRED`.
- **Resolution:** identical grounds to OI-08 — this *is* OI-08's decision, recorded separately in `02`'s §K because `02` split the task into "fix" (closed) and "parameterise" (escalated). The fix half is already `RESOLVED BY TECHNICAL EVIDENCE`; the parameterisation half is decided with OI-08.
- **Final status:** `CLOSED`.

### G.2 Reconciliation confirmations (§G.3 rows)

#### OI-14 — Reconciler cadence  → **CLOSED**

- **Source:** `TD-6-03` (cadence), `02` §G.3 row 7.
- **Previous status:** `RECOMMENDED — CONFIRMATION REQUIRED`.
- **Evidence reviewed:** `group-allotment/infrastructure/reconciliation/gba-reconciliation.service.ts:131` — `@Cron(CronExpression.EVERY_HOUR)`, report-only, hotel-scoped, gated by `gba.reconciliation.enabled`; `phase-5/04:924`; FDS §28.1 (flag row: *"hourly detectors"*); P-21 (flags default OFF).
- **Evidence result:** `PROVEN` the repository already carries an hourly, report-only, hotel-scoped, flag-gated reconciliation precedent with exactly the boundary Phase 6 needs.
- **Resolution:** Class (B). Cadence is a configuration value, not a contract. Decided: **hourly, matching the GBA precedent, behind a flag default OFF (P-21)**, parameterised so it can be tuned to whatever BD-6-05's soak adopts. `02`'s recommendation is confirmed as a decision.
- **Authority:** engineering (technical evidence).
- **Planning impact:** **BLOCKS NEITHER.**
- **Execution impact:** flag must be flipped as an operational action with gate evidence (G-5) — never in a commit.
- **Dependencies:** OI-04 (if the soak adopts a threshold, cadence becomes its measurement interval).
- **Required follow-up:** record cadence as a task attribute.
- **Final status:** `CLOSED`.

#### OI-15 — Where reconciliation findings persist  → **EXECUTION GATE ONLY**

- **Source:** `TD-6-03` (persistence), `02` §G.3 row 5.
- **Previous status:** `RECOMMENDED — CONFIRMATION REQUIRED` (*"REQ-25.3 boundary must be demonstrable"*).
- **Evidence reviewed:** FDS §26.2 four-layer table (`03:1030-1037`) — layer 3 "Reconciliation evidence: `/reconciliation` deltas, GBA detectors (`gba.reconciliation.enabled`, `automaticRepair:false`)", explicitly **not** authority; REQ-25.3 (`03:1020`) — may not feed decisions, may not write domain state; Deviation C / P-7 / L-r-23 — **no schema change is authorized**; FDS §26.1 (`03:1024-1026`) canonical log routes.
- **Evidence result:** `PROVEN` the **requirement** is already fully specified: findings belong in the reconciliation-evidence layer (§26.2 layer 3), hotel-scoped, outside domain state, with REQ-25.3 as the demonstrable acceptance boundary. `PROVEN` a new findings **table** is out (Deviation C). `UNKNOWN` — which existing non-domain mechanism will hold them, because that depends on the reconciler's concrete build, which does not exist yet.
- **Resolution:** Requirement = class (C), closed. Mechanism = class (B), but genuinely undeterminable until `TD-6-03` is designed under the no-schema-change constraint — that is an **execution-time** technical selection, not an owner decision and not an evidence gap.
- **Authority:** engineering, at execution.
- **Planning impact:** **BLOCKS NEITHER.** The plan states the acceptance criterion verbatim (findings persist in §26.2 layer 3, hotel-scoped, provably not feeding any decision or write — REQ-25.3 test).
- **Execution impact:** `GATE` — mechanism must be chosen and its REQ-25.3 boundary demonstrated before implementation.
- **Dependencies:** `TD-6-03`; Deviation C.
- **Required follow-up:** plan carries REQ-25.3 as an acceptance test.
- **Final status:** `EXECUTION GATE ONLY`.

---

## H. Planning-Blocker Assessment

Applying the test in D.3 to all 30 items:

### H.1 BLOCKS PLANNING

**None — with one authoring caveat.**

The single item touching the plan document itself is **OI-10** (document set / stage model / task-ID space). Under the literal test it does **not** block the definition of scope, dependency, behavior, verification, or execution order — `02` §R already enumerates scope (`TD-6-01`, `TD-6-03`, `TD-6-04`, `BD-6-06`, `TD-6-09`, `TD-6-10`, plus execution of `BD-6-04` and `BD-6-05`), and every dependency, behavior, verification method, and ordering constraint is recorded regardless of ID prefix. What it blocks is the **drafting conventions** of the plan document. It is therefore an authoring precondition, recorded as such, and is counted as an owner decision (§E), not as a planning blocker.

**Final `PLANNING BLOCKER` count: 0.**

### H.2 BLOCKS EXECUTION ONLY

OI-02, OI-03, OI-04, OI-07 *(capacity-grade evidence)*, OI-13, OI-15, OI-16, OI-18, OI-20, OI-21, OI-22, OI-23, OI-24, OI-25 — plus the ratified gates CO-6-01 (no legacy writer removal before E-1), FDS §24.3 (no premature cuts, BR-5-026), FDS §28.2/§28.4 (flag changes are operational actions with gate evidence, never in a commit), and `02` §H exit gates 1–9.

### H.3 BLOCKS NEITHER

OI-01, OI-05, OI-06, OI-08, OI-09, OI-10 *(authoring conventions only)*, OI-11, OI-12, OI-14, OI-17, OI-19, OI-26, OI-27, OI-28, OI-29, OI-30.

### H.4 Why the count is zero — the four hardest cases

1. **OI-03 (EOL date)** — a plan task reads *"retire external `engine/modify`; gate: E-5 returned or owner risk accepted; date: ⟨owner⟩"*. Scope, dependency, behavior, verification, order all defined.
2. **OI-04 (soak criteria)** — *"enable `canonicalRead`; gate: criteria recorded and met; then `canonicalWrite`"*. The criteria are a gate value, not a scope decision.
3. **OI-07 (read contract)** — a conditional task with **both** branches fully specified, including verification (one DB-backed test per surface with ASSERTION_MANAGED balances seeded) and rollback (annotate the gap; withhold capacity baselines).
4. **OI-18 (census)** — the census *is* plan scope (`TD-6-04` step one). An item that the plan exists to perform cannot block the plan.

---

## I. Execution-Only Gates

Items that are fully planned but may not **run** until a precondition holds. None requires a decision to author the plan.

| Gate | Item(s) | Precondition | Authority |
|---|---|---|---|
| Legacy writer removal / any cutover cut | OI-18, OI-19 | E-1 closes (census produced) | CO-6-01; FDS §24.3 BR-5-026 |
| DS-02 frontend cutover to authority numbers | OI-18, OI-20 | DS-01 operational (E-1 + E-3 + evaluator) | FDS §24.2 (`03:957-971`) |
| External `engine/modify` retirement; `TD-6-02` port/retire timing | OI-03, OI-13, OI-22 | E-5 returned **or** owner sets date accepting residual risk | BR-5-017/018/025; FDS §24.4; T5-88 |
| `gba.pickup.canonicalWrite` ON | OI-04 | `canonicalRead` ON + soak criteria met | FDS §28.3/§28.5 (P-21) |
| **Capacity-grade** cutover baselines | OI-07, OI-17 | TD-6-09 confirmed and, if approved, implemented with DB-backed proof | `02` §H exit gate 3 |
| Reconciler build (`TD-6-03`) | OI-15 | findings mechanism chosen; REQ-25.3 boundary demonstrable; no schema change | REQ-25.3; Deviation C |
| Counter-vs-balance comparator build | OI-13, OI-17 | `TD-6-03` design | B-1/B-2/B-3; REQ-25.1 |
| Population assignment tooling | OI-18 | census denominator exists | D-21; CO-6-02 |
| Any forward/backward flag action | OI-20 | gate evidence recorded; never inside a commit | G-5; FDS §24.8.4; §28.2 |
| Every code-changing exit | OI-25 | E-8 reconciled executed baselines | FDS §31 E-8 |
| `BD-6-02` / `TD-6-05` long-term resolution | OI-02, OI-16 | OI-18 census; then owner decision if a live writer is evidenced | BR-5-046 |
| Route merge (`TD-6-06`) | OI-24, OI-27 | E-7 observed | FDS §31 E-7 |

---

## J. Deferred Items

Deferrals confirmed as **correct and ratified** — none is a disguised decision.

| ID | Item | Deferral authority | Why deferral is right |
|---|---|---|---|
| OI-05 | `T5-36` client math / allowlist | `02` §R (not in plan scope) | GBA/frontend display hygiene; unrelated to reconciliation, cutover, or retirement. Owed by another workstream. |
| OI-06 | BLK-1 / BLK-2 wash | Phase 5 plan §32–§33; `02` X-5 | Phase 4 carry-over; Phase 5 was forbidden to execute it; sole effect is gating an already-false flag. |
| OI-11 | `T5-11` interim-gate test (question S-7) | Phase 5 ownership; `02` §J | Phase 5 is closed and owns the task. Phase 6 neither re-opens nor retro-closes it, and flag changes are operational (CO-6-06), not Phase 6 work. **Recorded for the lead's convenience:** the honest-gate property is separately evidenced by T5-08 (22 tests incl. conflict/absence/failure), T5-09 binding swap, and the fail-closed family — so if the question is ever asked, the answer is already available. |
| OI-27 | Duplicate `GET /tax-rates` merge | BR-5-044 hygiene; `02` `DEFERRED` | Non-availability hygiene; gated by OI-24 (E-7). |
| OI-28 | `gba.reconciliation.enabled` / `twoLayerConsult` activation | `02` `DEFERRED` | No ratified activation order. If BD-6-05 adopts the parity-threshold soak, `gba.reconciliation.enabled` becomes a **precondition** of that soak — a sequencing observation, not a new rule. |
| OI-29 | Allowlist / dead-DI removal timing (G-14/G-15/G-16) | `02` `DEFERRED` | Each removal rides with the retirement it accompanies; early removal breaks the scan gates Phase 5 uses as its retirement-candidate register. |
| OI-30 | Analytics contract / channel publication | **P-20, P-19** (ratified) | Outbound push and analytics computation are permanently out of availability scope. Not a pending decision. |

**No other item is deferred.**

---

## K. Final Resolution Matrix

| ID | Source | Final status | Planning impact | Authority | Counted in |
|---|---|---|---|---|---|
| OI-01 | BD-6-01 | **`CLOSED`** | neither | ratified rules | CLOSED |
| OI-02 | BD-6-02 | **`EVIDENCE REQUIRED`** | execution only | census → BR-5-046 (→ owner only if live writer evidenced) | EVIDENCE |
| OI-03 | BD-6-03 | **`OWNER DECISION REQUIRED`** | neither (authoring) / execution gate | product owner | OWNER |
| OI-04 | BD-6-05 | **`OWNER DECISION REQUIRED`** | neither (authoring) / execution gate | product-ops owner | OWNER |
| OI-05 | BD-6-07 | **`DEFERRED`** | neither | product owner (other workstream) | DEFERRED |
| OI-06 | BD-6-08 | **`DEFERRED`** | neither | product owner (Phase 4 carry-over) | DEFERRED |
| OI-07 | TD-6-09 | **`OWNER DECISION REQUIRED`** | neither (both branches specified) | engineering owner + product acknowledgement | OWNER |
| OI-08 | TD-6-10 | **`CLOSED`** | neither | engineering (technical evidence) | CLOSED |
| OI-09 | TD-6-14 | **`CLOSED`** | neither | engineering (technical evidence) | CLOSED |
| OI-10 | TD-6-15 | **`OWNER DECISION REQUIRED`** | authoring conventions only | phase owner | OWNER |
| OI-11 | TD-6-08 / T5-11 | **`DEFERRED`** | neither | Phase 5 ownership | DEFERRED |
| OI-12 | TD-6-01 (param.) | **`CLOSED`** | neither | engineering (technical evidence) | CLOSED |
| OI-13 | TD-6-02 (timing) | **`EXECUTION GATE ONLY`** | execution only | ops inventory + BR-5-017 | EXECUTION |
| OI-14 | TD-6-03 (cadence) | **`CLOSED`** | neither | engineering (technical evidence) | CLOSED |
| OI-15 | TD-6-03 (persistence) | **`EXECUTION GATE ONLY`** | execution only | engineering at execution | EXECUTION |
| OI-16 | TD-6-05 (long-term) | **`EVIDENCE REQUIRED`** | execution only | census → BR-5-046 | EVIDENCE |
| OI-17 | CO-6-04 | **`CLOSED`** | neither | see K.1 | CLOSED |
| OI-18 | E-1 (population) | **`EVIDENCE REQUIRED`** | execution only | read-only census (TD-6-04) | EVIDENCE |
| OI-19 | E-1 (writer) | **`CLOSED`** | neither | repository evidence | CLOSED |
| OI-20 | E-3 | **`EVIDENCE REQUIRED`** | execution only | runtime observation | EVIDENCE |
| OI-21 | E-4 | **`EVIDENCE REQUIRED`** | neither | product/domain definition | EVIDENCE |
| OI-22 | E-5 | **`EVIDENCE REQUIRED`** | neither (T5-88 disposition rule) | ops/gateway inventory | EVIDENCE |
| OI-23 | E-6 | **`EVIDENCE REQUIRED`** | neither | runtime observation | EVIDENCE |
| OI-24 | E-7 | **`EVIDENCE REQUIRED`** | neither | runtime trace | EVIDENCE |
| OI-25 | E-8 | **`EXECUTION GATE ONLY`** | execution only | executed test runs | EXECUTION |
| OI-26 | E-2 | **`NOT APPLICABLE`** (conditional) | neither | BR-5-003 standing | NOT APPLICABLE |
| OI-27 | TD-6-06 | **`DEFERRED`** | neither | engineering hygiene | DEFERRED |
| OI-28 | TD-6-11 | **`DEFERRED`** | neither | P-21 sequence | DEFERRED |
| OI-29 | TD-6-13 | **`DEFERRED`** | neither | retirement-coupled | DEFERRED |
| OI-30 | BD-6-09/10 | **`DEFERRED`** | neither | **P-19, P-20** (ratified) | DEFERRED |

### K.1 The one correction to `02` — OI-17 (CO-6-04 / §G.3 comparator sufficiency)

`02` recorded: *"`expected` inherits the read omission … `DIAGNOSTIC_VARIANCE` readings are not trustworthy as cutover evidence until this is resolved"*, and set a blanket sequencing rule — *"confirm TD-6-09 before capturing **any** cutover baseline."*

**Current evidence refines this.** The claim implicitly assumes the **legacy** side of the comparison does *not* inherit the same omission. It does:

- The only live in-repo writer of the legacy `availability` counters is the **external** modify path — `crs-engine.service.ts:391-402` (`inventoryDomain.reserve` / `.release`), which is M-8. `inventory.domain-service.ts:164,181` holds the raw `UPDATE availability … reserved = …, available = …` statements; its only other would-be caller, `reservation.repository.ts:89`, **injects** `InventoryDomainService` but **never invokes it** (verified: `:15` import, `:89` constructor injection, zero call sites). `reserveRooms` has no callers at all.
- Authority-created reservations therefore write **assertion balances**, not legacy counters.
- Consequently for a stay with N ASSERTION_MANAGED reservations and no legacy activity: legacy `available` = capacity **and** canonical `sellableAvailable` = capacity — both omit the same N, and `/reconciliation` reports `MATCH`.

**Therefore:**

1. `MATCH` does **not** mean "correctly sellable" — absolute capacity is off by N. `02`'s warning about interpreting the comparator as *capacity* evidence is **correct and stands**.
2. But `VARIANCE` readings remain **trustworthy as projection-drift evidence**, which is precisely what REQ-25.1 sanctions (`/reconciliation` is the legacy-vs-authority comparison, report-only). The comparator is **like-for-like**, not biased.
3. What cannot be obtained from this comparator — assertion-side integrity — was never available from it: **I-2 is ABSENT**, and that is `TD-6-03`'s job, not `TD-6-09`'s.

**Refined sequencing rule (replaces `02`'s blanket rule):**

> *Projection-drift* baselines (the comparator's ratified purpose) may be captured **now**.
> *Absolute sellable-capacity* baselines require **OI-07 confirmed and, if approved, implemented** first.
> *Assertion-integrity* evidence requires **I-2 built** (OI-15 / `TD-6-03`), which is a separate comparator and was always absent.

**Resolution:** the question *"is the current comparator sufficient for cutover evidence?"* now has a **two-tier answer** rather than a blocking "not until TD-6-09". **`CO-6-04` is closed** as a decision; OI-07 remains an owner decision on its own merits (§E), and the capacity-tier gate is carried in §I.

- **Authority:** repository evidence (M-8 verified above) + REQ-25.1.
- **Final status:** `CLOSED`.

---

## L. Implementation Planning Readiness

### L.1 Counts

| Final status | Count |
|---|---|
| `CLOSED` | **7** (OI-01, OI-08, OI-09, OI-12, OI-14, OI-17, OI-19) |
| `OWNER DECISION REQUIRED` | **4** (OI-03, OI-04, OI-07, OI-10) |
| `EVIDENCE REQUIRED` | **8** (OI-02, OI-16, OI-18, OI-20, OI-21, OI-22, OI-23, OI-24) |
| `PLANNING BLOCKER` | **0** |
| `EXECUTION GATE ONLY` | **3** (OI-13, OI-15, OI-25) |
| `DEFERRED` | **7** (OI-05, OI-06, OI-11, OI-27, OI-28, OI-29, OI-30) |
| `NOT APPLICABLE` | **1** (OI-26) |
| **Total reviewed** | **30** |

### L.2 What changed versus `02`

- **3 items left the owner queue on evidence/rule grounds:** BD-6-01 (authority half ratified by BR-5-045 + FDS §13.3; lifecycle half needs no Phase 6 decision — no rule requires removal before Phase 11, so the default is KEEP read-only to Phase 11, with earlier removal available as an *optional* hygiene task the plan will not wait for); TD-6-10 and TD-6-01's parameterisation half (class B technical scope, no business consequence); TD-6-14 (class B — scheduled poll satisfies every REQ-25.x duty with no new artifact).
- **2 items moved out of Phase 6 scope entirely:** BD-6-07 and BD-6-08 remain owed to their owning workstreams but are not Phase 6 decisions and do not gate Phase 6.
- **E-1 split and half closed:** writer identification is now `PROVEN` for all five store families (OI-19 `CLOSED`); only population/recency remains, with the read-only census already registered as `TD-6-04`'s first step.
- **E-5 assessed for planning impact from a ratified rule, not an opinion:** T5-88 (`04:261,272`) makes it non-blocking for disposition; local gateway evidence (OI-22) records that no consumer registry exists in-repo and states plainly what that does not prove.
- **1 correction issued:** `CO-6-04`'s blanket "confirm TD-6-09 before capturing **any** baseline" is refined to a two-tier rule (§K.1), because M-8 shows the legacy counter side inherits the same omission, making the comparison like-for-like for its ratified purpose.
- **BD-6-05 confirmed genuinely open:** four independent locations preserve the soak *sequence* with no definition anywhere; the only defined soak (T5-10) is a different artifact. Not inferable — left with the owner.

### L.3 Readiness verdict

> ## `READY FOR IMPLEMENTATION PLANNING WITH GATED ITEMS`

**Grounds.**

1. **No item blocks the plan's ability to define scope, dependency, behavior, verification, or execution order.** `PLANNING BLOCKER = 0` under the literal test (§H.1), including the four hardest cases argued in §H.4.
2. **Scope is enumerated and closed** (`02` §R): `TD-6-01`+`TD-6-10` (one bundled task), `TD-6-03` (scheduled report-only reconciler + I-2 comparator), `TD-6-04` (census then first-touch tooling under D-21), `BD-6-06`/`T5-59` (typed delete-reject — ratified, implementation required), `TD-6-09` (conditional, both branches specified), plus execution of `BD-6-04` and `BD-6-05` once their gates are set. **Nothing may enter the plan without a decision ID.**
3. **Every remaining open item is expressible as a gated task or an evidence task** — 4 owner decisions and 8 evidence items, all with named preconditions and named authorities (§E, §F, §I).
4. **Phase 5 remains closed** (12/12 gates) and nothing here re-opens, retro-closes, or amends it.

**Conditions attached to the verdict:**

- **C1 (precedes drafting):** the phase owner settles **OI-10** — document set, stage model, task-ID space — before the Implementation Plan document is authored. This is the only item whose answer is needed to *start writing*.
- **C2 (gates inside the plan):** **OI-03**, **OI-04**, **OI-07** are recorded as gate values on their respective tasks. The plan may be authored and reviewed before they are answered.
- **C3 (gates before execution):** the §I table is reproduced as the plan's precondition register. In particular: E-1 census before any legacy writer removal (CO-6-01); TD-6-09 confirmed before any *capacity-grade* baseline; E-5 or an owner-set date before external retirement; no flag changed inside a commit (G-5).
- **C4 (scope hygiene):** OI-05, OI-06, OI-11, OI-27, OI-28, OI-29, OI-30 must **not** enter the Phase 6 plan. They are other workstreams' items, deferred by ratified authority.
- **C5 (no back-door work):** nothing in §E–§K authorises code, schema, migration, test, flag flip, reconciliation run, cutover, or legacy deletion. Each "Implementation required" statement is a plan instruction, not an action taken here.

**What would flip this verdict to `NOT READY FOR IMPLEMENTATION PLANNING`:** an item in §H.1. There is none. Two conditions would create one — a phase owner declining to settle OI-10 (then no plan document can be authored), or the census (OI-18) returning evidence that forces `BD-6-02`/`TD-6-05` into a scope-affecting owner decision that cannot be expressed as a conditional task.

---

**End of `02A_OPEN_DECISION_EVIDENCE_RESOLUTION.md`.**
No code, schema, migration, test, flag flip, reconciliation run, cutover, legacy deletion, or Implementation Plan was created by this document.

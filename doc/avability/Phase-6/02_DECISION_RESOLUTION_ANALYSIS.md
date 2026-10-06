# XYLO Availability Phase 6 — Decision & Resolution Analysis

**Phase:** 6 — Reconciliation, Cutover, and Legacy Retirement (decisions only — NOT implementation)
**Date:** 2026-10-06
**Inputs:** `docs/availability/phase-6/01_FORENSIC_AUDIT.md` (authoritative audit baseline); `docs/availability/phase-5/03_FINAL_DOMAIN_SPECIFICATION.md`, `04_IMPLEMENTATION_PLAN.md`, `06_EXECUTION_EVIDENCE.md`; `docs/availability/phase-4/05_BUSINESS_RULES_CURRENT_STATE.md`, `07_LEGACY_DEPENDENCY_MAP.md`, `11_TARGET_BUSINESS_RULES.md`, `12_DECISION_STATUS.md`, `13_FINAL_DOMAIN_SPECIFICATION.md`, `17_PHASE4_READINESS_REVIEW.md`; `docs/enterprise/availability-phase3-implementation-plan.md`, `availability-phase3-domain-specification.md`; targeted read-only source verification in `apps/api/src/modules/{availability,reservations,rates-inventory,activities,front-office}` and `apps/web`.
**Scope honored:** 0 code, 0 schema, 0 migration, 0 backfill, 0 test change, 0 flag flip, 0 reconciliation run, 0 cutover action, 0 legacy deletion, 0 Implementation Plan. Every statement below is either a citation or a disposition. No business rule is ratified here.

---

## 0. Method and Conventions

### 0.1 Status vocabulary (used verbatim)

| Status | Meaning |
|---|---|
| `RESOLVED BY EXISTING DOMAIN RULE` | A ratified XYLO rule already decides the question. Phase 6 **preserves** it; it may not be re-opened, reinterpreted, or silently replaced. |
| `RESOLVED BY TECHNICAL EVIDENCE` | Repository, schema, or executed-evidence facts force a single answer. No policy choice remains. |
| `RECOMMENDED — CONFIRMATION REQUIRED` | Evidence narrows to one best option, but the choice has product-visible consequence or touches a published contract; owner must sign off. |
| `REQUIRES OWNER DECISION` | A genuine business/domain policy choice. **Ratification is prohibited in this document**; recorded with options and consequences only. |
| `EVIDENCE GAP` | Cannot be decided with the evidence available in this repository. Carried as `UNKNOWN` / `EVIDENCE INSUFFICIENT`, never guessed. |
| `UNRESOLVED — REQUIRES EXPLICIT DECISION` | Flagged, deliberately not decided here (scope or authority). |
| `DEFERRED` | Belongs to a later phase or a later activation gate, with the authority named. |
| `NO ACTION REQUIRED` | Verified correct, sanctioned as-is, or outside the Phase 6 boundary fence. |

### 0.2 Decision record format

Every decision in §D/§E carries: **ID → Source finding → Exact evidence → Current behavior → Problem → Decision question → Options → Option consequences → Recommended resolution → Final decision → Decision authority → Ratified rule or technical invariant → Acceptance criteria → Dependencies → Downstream impact → Verification requirements.** Where a field cannot be evidenced it is written `UNKNOWN` or `EVIDENCE INSUFFICIENT`.

### 0.3 Evidence and authority hierarchy (unchanged from the brief)

1. Local repository tree (code, schema, `.env`).
2. `docs/availability/phase-6/01_FORENSIC_AUDIT.md` — the audit of record for this phase.
3. Authoritative availability domain documents (`phase-5/03_…`, `phase-4/11_…`, `phase-4/13_…`).
4. Phase 5 closed evidence (`phase-5/06_…`) — **input, not re-audit subject**.
5. Phase 4 specifications and decision registers.
6. Local schema / DB state.

**GitHub is not implementation authority.** No remote fetch was performed and none is required (`01_FORENSIC_AUDIT.md` §P.1 P-9).

**Decision-authority split.** (A) facts → repo/DB/docs; (B) technical decisions resolvable from evidence → decided here as `RESOLVED BY TECHNICAL EVIDENCE`; (C) existing ratified rules → preserved verbatim, never redefined; (D) new business rules → `REQUIRES OWNER DECISION` only.

### 0.4 Phase 5 closure

Phase 5 is **CLOSED** (12/12 exit gates PASS, `06_EXECUTION_EVIDENCE.md:2088-2092`). This document does **not** reopen it, does **not** restart the Phase 5 audit, and does **not** invent Phase 6 stages, workflow, or document trees (audit §C.4 D-4/D-5). Where Phase 5 evidence is incomplete (§J), the disposition is a Phase 6 decision about *what happens next*, not a re-litigation of Phase 5.

### 0.5 Section map

| Brief section | Heading |
|---|---|
| A | `## A. Purpose and Scope` |
| B | `## B. Audit Baseline` |
| C | `## C. Decision Inventory` |
| D | `## D. Business Decisions` |
| E | `## E. Technical Decisions` |
| F | `## F. Entry Blocker Matrix` |
| G | `## G. Reconciliation Decisions` |
| H | `## H. Cutover Decisions` |
| I | `## I. Legacy Retirement Dispositions` |
| J | `## J. Phase 5 Closure Dispositions` |
| K | `## K. Requiring Business Confirmation` |
| L | `## L. Resolved by Existing Domain Rule` |
| M | `## M. Resolved by Technical Evidence` |
| N | `## N. Deferred` |
| O | `## O. No Action Required` |
| P | `## P. Required Rules and Invariants to Preserve` |
| Q | `## Q. Acceptance Criteria` |
| R | `## R. Downstream Implications` |
| S | `## S. Open Questions` |
| T | `## T. Final Decision Summary` |
| (brief §15) | `## U. Methodology Review — retained / modified / reconsidered` |

---

## A. Purpose and Scope

**Purpose.** Convert every finding, contradiction, divergence, gap, flag, legacy item and blocker recorded in `01_FORENSIC_AUDIT.md` into an explicit disposition, using exactly the classifications the brief allows, so that a subsequent Phase 6 Implementation Plan (not created here) has a complete and closed decision input.

**In scope.** Classification and resolution of audit findings; preservation of ratified rules; identification of decisions that require a human owner; acceptance criteria and downstream implications; disposition of the four unevidenced Phase 5 tasks; legacy REMOVE/KEEP/DEFER confirmation; reconciliation and cutover invariants.

**Out of scope (prohibited by the brief and/or the Phase 6 boundary fence).** Code changes; schema changes or migrations; test changes; flag flips; reconciliation runs; cutover execution; legacy deletion; creation of an Implementation Plan; invention of a Phase 6 stage model, task-ID space, or document tree (audit §C.4 D-4/D-5); reopening Phase 5; ratification of new business rules.

---

## B. Audit Baseline

### B.1 Source of record

`docs/availability/phase-6/01_FORENSIC_AUDIT.md` is the sole finding baseline. Findings are referenced by their native IDs throughout this document — no finding is renumbered, merged, or dropped.

### B.2 Phase 6 scope of record (quoted, not reinterpreted)

| Ref | Statement | Source |
|---|---|---|
| B-1 | "Cutover of a Reservation population to assertion authority, deterministic population migration at scale, legacy counter retirement, dual-source reconciliation closure, and removal of `crs-engine` inventory calls." | `docs/enterprise/availability-phase3-implementation-plan.md:427` (§13.4) |
| B-2 | "Deferred to Phase 6 \| population cutover, legacy counter retirement, dual-source reconciliation." | same file `:442` |
| B-3 | Legacy `availability`/`inventory` counter retirement → Phase 6; population cutover at scale; dual-source reconciliation closure → Phase 6; "Removing `crs-engine` inventory calls from untouched legacy paths" → Phase 6 | same file `:644-646` |
| B-4 | "Reconciliation, population cutover, and legacy retirement belong to Phase 6; this specification does not perform them." | `availability-phase3-domain-specification.md:445` |
| B-5 | "Legacy counter cutover — RESOLVED — deferred to Phase 6" (D-20) | `availability-phase3-business-rules-decision-sheet.md:22` |
| B-6 | A1 is sole sellable-capacity authority — "gates Phase 6 integration & D-7" (D-6) | `phase-4/12_DECISION_STATUS.md:17` |
| B-7 | `room_inventory` disposition "KEEP read-only or REMOVE (Phase 6)" (L-10) | `phase-4/07_LEGACY_DEPENDENCY_MAP.md:36` |
| B-8 | Seven Phase-4 carry-overs Phase 5 "may not assume … and may not execute" — all still open | `phase-5/04_IMPLEMENTATION_PLAN.md:1410-1425` (§32) |
| B-9 | Ten explicitly deferred items, each with authority | same file `:1429-1442` (§33) |
| B-10 | Seven non-blocking notes carried out of Phase 5 | `phase-5/06_EXECUTION_EVIDENCE.md:2100-2118` (§34) |

### B.3 Boundary fence

**Inside Phase 6:** population cutover; legacy counter retirement; dual-source reconciliation closure; `crs-engine` inventory-call removal; L-10 `room_inventory` disposition; B-8 carry-over dispositions; B-9 items tagged Phase 6/11.

**Explicitly not Phase 6:** schema authoring (Deviation C / G-1 / P-7); deletion of the legacy `availability` table (P-16 → Phase 11); the ten B-9 items naming P-19/P-20/P-21/DS-06/DS-07/DS-08/F-14; ADR-072 reservation semantics; admin/mobile availability surfaces (P-19/M-7).

### B.4 Finding registers carried into this document

| Register | Range | Count | Dispositioned in |
|---|---|---|---|
| Entry blockers | X-1…X-7 | 7 | §F |
| Business decisions | O-1…O-10 | 10 | §D, §K |
| Technical decisions | O-11…O-18 | 8 | §E, §M |
| Divergences | H-1…H-7 | 7 | §D (H-3/H-4), §E (H-1/H-2/H-5/H-6/H-7) |
| Reconciliation machinery | I-1…I-7 | 7 | §G |
| Flags | J-1…J-7 + J-AUD-01 | 8 | §H |
| Legacy inventory | G-1…G-16 | 16 | §I |
| Evidence coverage | L-1…L-9 | 9 | §J |
| Contradictions | M-1…M-8 | 8 | §J, §M |
| Defects | N-1…N-6 | 6 | §E |
| Not-required work | P-1…P-9 | 9 | §O |
| Doc drift | C-5.1…C-5.5 | 5 | §M, §S |
| Findings raised by this analysis | D-AUD-01…D-AUD-03 | 3 | §E, §G, §M |

---

## C. Decision Inventory

Master register. `Final decision` is the status assigned by this document; it is **not** an implementation authorization.

### C.1 Business / domain decisions

| ID | Source | Subject | Final decision | Authority |
|---|---|---|---|---|
| BD-6-01 | O-1, G-5, B-7 | `room_inventory` — keep read-only or remove | `REQUIRES OWNER DECISION` (recommend: KEEP read-only to Phase 11) | Product/operations owner |
| BD-6-02 | O-2, H-3, H-4, X-4, G-8 | `rate_restrictions` as a hard quote blocker while authority reports `UNRESOLVED` | Transitional state `RESOLVED BY EXISTING DOMAIN RULE`; steady state `UNRESOLVED — REQUIRES EXPLICIT DECISION`, contingent on E-1 | Product owner + E-1 evidence |
| BD-6-03 | O-3, N-2, H-1, G-3, X-1 | External `POST /rates/engine/modify` end-of-life date | `EVIDENCE GAP` (E-5) then `REQUIRES OWNER DECISION` | Product owner + external inventory |
| BD-6-04 | O-4, J-2, N-3, M-5, X-6 | `gba.consumers.cascade` deploy-invariant timing | `RESOLVED BY EXISTING DOMAIN RULE` (P-21) — operational execution outstanding | Operational release owner |
| BD-6-05 | O-5, J-6, J-7, X-6 | `canonicalRead` → soak → `canonicalWrite` ordering **and** soak exit criteria | Order `RESOLVED BY EXISTING DOMAIN RULE`; soak definition `REQUIRES OWNER DECISION` | Product/ops owner |
| BD-6-06 | O-6, L-7, M-1 | Typed delete-reject (BR-5-043) vs raw FK rejection | `RESOLVED BY EXISTING DOMAIN RULE` → `DIRECT TECHNICAL CORRECTION REQUIRED` | Engineering (ratified rule exists) |
| BD-6-07 | O-7, L-6, G-14 | GBA `pickupPct` client math — remove (T5-36) or sanction permanently | `REQUIRES OWNER DECISION` (scope) | Product owner |
| BD-6-08 | O-8, X-5 | BLK-1 (GUARANTEED_BLOCK wash exclusion) and BLK-2 (durable wash/release/attrition store) | `DEFERRED` to wash activation + `REQUIRES OWNER DECISION` | Product owner (Phase 4 carry-over) |
| BD-6-09 | O-9, P-20 | Availability analytics / canonical occupancy contract | `DEFERRED` (P-20, DS-06) | Backlog |
| BD-6-10 | O-10, P-19 | Outbound channel/CRS push publication semantics | `DEFERRED` (P-19, DS-07) | Backlog |

### C.2 Technical decisions

| ID | Source | Subject | Final decision | Authority |
|---|---|---|---|---|
| TD-6-01 | O-11, N-1 | `rtFilterRt` alias defect on `GET /availability/matrix` | `RESOLVED BY TECHNICAL EVIDENCE` — fix required; parameterisation `RECOMMENDED — CONFIRMATION REQUIRED` | Engineering |
| TD-6-02 | O-12, N-2, H-1, H-7 | `crs.modifyReservation`: consult the assertion port, or retire | `EVIDENCE GAP` (E-5) gates the answer; disposition `RESOLVED BY EXISTING DOMAIN RULE` (BR-5-017/§21.4 = retire) | Engineering + BD-6-03 |
| TD-6-03 | O-13, I-1, I-2, N-4, I-6, I-7 | Reconciliation shape: persistence, cadence, repair | `RESOLVED BY EXISTING DOMAIN RULE` for boundary (report-only, never repair); mechanism `RESOLVED BY TECHNICAL EVIDENCE` | Engineering |
| TD-6-04 | O-14, I-3 | Population cutover mechanism at scale | `RESOLVED BY EXISTING DOMAIN RULE` (D-21 first-touch, no bulk backfill) + tooling `RESOLVED BY TECHNICAL EVIDENCE` | Engineering |
| TD-6-05 | O-15, H-3, G-8 | `evaluateRestrictions` — authority consult vs `rate_restrictions` read | `EVIDENCE GAP` (E-1); interim `RESOLVED BY EXISTING DOMAIN RULE` (fail-closed, no migration) | Engineering + E-1 |
| TD-6-06 | O-16, N-5, E-7 | Duplicate `GET /tax-rates` route merge | `DEFERRED` (non-availability hygiene; BR-5-044 duty recorded) | Engineering hygiene |
| TD-6-07 | O-17, M-3, C-5.3 | Phase 3-era legacy register — re-baseline or annotate | `RESOLVED BY TECHNICAL EVIDENCE` — annotate + re-verify, never rewrite | Engineering |
| TD-6-08 | O-18, L-3, L-4, L-6, L-7, M-1, X-2 | Disposition of `T5-11`, `T5-36`, `T5-41`, `T5-59` | Split disposition — see §J | Engineering lead + product (for BD-6-07 only) |
| TD-6-09 | **D-AUD-01** (new) | Canonical read omits assertion-balance consumption | `RECOMMENDED — CONFIRMATION REQUIRED` | Engineering owner |
| TD-6-10 | N-6 | Raw SQL identifier/filter interpolation on the A3 surface | `RECOMMENDED — CONFIRMATION REQUIRED` (bundle with TD-6-01) | Engineering |
| TD-6-11 | J-4, J-5, X-6 | Activation of `gba.reconciliation.enabled` and `gba.pickup.twoLayerConsult` | `DEFERRED` — unscheduled by any ratified sequence | Engineering + BD-6-05 |
| TD-6-12 | M-4, C-5.4, **D-AUD-02** (new) | Stale line anchors and stale prose in upstream documents | `RESOLVED BY TECHNICAL EVIDENCE` — re-verify at execution; annotate, do not rewrite closed evidence | Engineering |
| TD-6-13 | G-14, G-15, G-16 | Timing of allowlist / dead-DI removals | `DEFERRED` — each removal is gated by the retirement it accompanies | Engineering |
| TD-6-14 | I-6 | Missing assertion-lifecycle outbox event | `UNRESOLVED — REQUIRES EXPLICIT DECISION` (event vs poll vs none) | Engineering owner |
| TD-6-15 | X-7, §C.4 D-4/D-5 | Phase 6 document set, stage model, task-ID space | `UNRESOLVED — REQUIRES EXPLICIT DECISION` — deliberately **not** decided in this document | Phase owner |

### C.3 Classification roll-up

| Classification | Decision IDs |
|---|---|
| `RESOLVED BY EXISTING DOMAIN RULE` | BD-6-02 (transitional), BD-6-04, BD-6-06, BD-6-09, BD-6-10, TD-6-02 (disposition), TD-6-03 (boundary), TD-6-04 (mechanism), TD-6-05 (interim) |
| `RESOLVED BY TECHNICAL EVIDENCE` | TD-6-01, TD-6-04 (tooling), TD-6-07, TD-6-12 |
| `REQUIRES BUSINESS CONFIRMATION` (see §K) | BD-6-01, BD-6-02 (steady state), BD-6-03, BD-6-05 (soak), BD-6-07, BD-6-08, TD-6-09, TD-6-10 |
| `REQUIRES TECHNICAL DECISION` (explicit) | TD-6-14, TD-6-15 |
| `EVIDENCE GAP` | BD-6-03 (E-5), TD-6-02 (E-5), TD-6-05 (E-1), plus E-1…E-8 as registered |
| `DIRECT TECHNICAL CORRECTION REQUIRED` | TD-6-01 (fix), BD-6-06 → T5-59 implementation, D-AUD-01 (subject to confirmation) |
| `DEFERRED` | BD-6-08, BD-6-09, BD-6-10, TD-6-06, TD-6-11, TD-6-13 |
| `NO ACTION REQUIRED` | see §O |

---

## D. Business Decisions

> Ratification is prohibited here. Each record states options, consequences, and a recommendation only.

### BD-6-01 — `room_inventory` disposition (O-1)

- **Source finding:** `01_FORENSIC_AUDIT.md` G-5, O-1, B-7.
- **Exact evidence:** model `packages/db/schema.prisma:10413`; readers `apps/api/src/modules/activities/availability-sales.controller.ts:76,86,116,276` (feeds `availableRooms`); **no writer anywhere in the repo** (G-5); prohibition verified clean by T5-44 (`06_EXECUTION_EVIDENCE.md:1981-1993`); web scan `apps/web/features/availability/__tests__/t544-room-inventory-display.test.ts:38-95` — no UI renders the figure as availability.
- **Current behavior:** read-only, joins into the legacy matrix payload, exposed as `availableRooms`, **consumed by no view**.
- **Problem:** an unread, unwritten store still sits on the A3 read path; Phase 4 L-10 explicitly reserved the disposition for Phase 6.
- **Options:**
  - **A. KEEP read-only to Phase 11** (matches the P-16 treatment of every other legacy store).
  - **B. REMOVE now** (drop the two joins and the `available_rooms` payload field).
- **Consequences:** A preserves rollback and P-16 uniformity and defers a payload-shape change to the same wave as the other retirements; B simplifies the A3 query but changes a live response field and must also remove `reservation.api.ts:269`.
- **Recommendation:** **A** — P-16 (`phase-5/03_…:110`) already fixes legacy counters as "retained, read-only, never authoritative … deletion = Phase 11"; B would be a payload change with no functional gain while T5-44 proves zero display impact either way.
- **Final decision:** `REQUIRES OWNER DECISION`.
- **Authority:** Product/operations owner (B-7 assigns it to Phase 6 by name).
- **Ratified rule:** P-16; INV-P5-31; BR-5-045/046.
- **Acceptance criteria (whichever option):** no UI path renders `room_inventory` availability (existing `t544` gates stay green); `rg room_inventory` in non-test API source returns only the sanctioned sites if kept, or zero if removed.
- **Dependencies:** none. **Downstream impact:** A3 matrix query; `GET /room-types` payload.
- **Verification:** re-run `t544-room-inventory-display.test.ts` and `t537-shadow-fake-defaults.test.ts`.

### BD-6-02 — `rate_restrictions` epistemic status (O-2)

- **Source finding:** H-3, H-4, X-4, G-8, O-2.
- **Exact evidence:** hard-blocker read `crs-engine.service.ts:146-196` (used at `:245` and `:375`); authority read `prisma-restriction.adapter.ts:114,153` marks the store `unproven` → `UNRESOLVED`, never `BLOCKED`; **no in-repo writer** for `rate_restrictions` (`schema.prisma:8925`, G-8); composite truth `crs-engine.service.ts:244-288` (`blocked` from legacy `:250`, `available`/`eligibility` from authority `:282`).
- **Current behavior (transitional, deliberately):** T5-14 re-pointed the **availability** signal to the authority while leaving `blocked`/`blockReasons`/`restrictions` on exact legacy semantics, citing "§20 `engine/restrictions` retirement is a separate task" and BR-5-046 (rate context deliberately **omitted** from the authority consult, because a rate code would force every rate-coded quote to `UNRESOLVED`) — `06_EXECUTION_EVIDENCE.md:1320-1321`.
- **Problem:** a guest can be told "blocked" by a source the authority itself declines to trust, while the authority simultaneously reports `UNRESOLVED`.
- **Options:**
  - **A. Keep the transitional split until E-1 proves the writer**, then migrate `rate_restrictions` into the evaluator (BR-5-046 flags it) or exclude it permanently.
  - **B. Remove the legacy hard-blocker now** — quotes then never say `blocked` from `rate_restrictions`; risk = silent underselling if a real writer exists.
  - **C. Force fail-closed everywhere** — treats `rate_restrictions` as `UNRESOLVED` and refuses quotes until the writer is known.
- **Consequences:** A preserves current behavior and is already ratified as the transitional shape; B risks underselling against a live restriction; C blocks all rate-coded selling (T5-14 explicitly rejected this for the authority consult).
- **Recommendation:** **A**, contingent on E-1. Change neither leg before the writer population is known.
- **Final decision:** transitional state `RESOLVED BY EXISTING DOMAIN RULE` (T5-14 + BR-5-016/017/046/050, FDS §20/§21.4); steady state `UNRESOLVED — REQUIRES EXPLICIT DECISION` pending E-1.
- **Authority:** Product owner for the steady-state rule; E-1 evidence gate.
- **Ratified rule:** BR-5-046 (flag/exclude unproven stores), BR-5-003 (conflict ⇒ `UNRESOLVED`), BR-5-016/017/050, FDS §20.
- **Acceptance criteria:** whatever is chosen, `available = eligibility === 'ELIGIBLE' && !blocked` remains the single composite contract (`use-crs-book.ts` gate), and no response field mixes two numbers without provenance.
- **Dependencies:** E-1. **Downstream:** `getQuote`, `evaluateRestrictions`, `engine/restrictions` retirement (already 404 — P-4).
- **Verification:** per-hotel writer census for `rate_restrictions` (the E-1 artifact) before any change.

### BD-6-03 — External `POST /rates/engine/modify` end-of-life (O-3)

- **Source finding:** O-3, N-2, H-1, G-3, X-1.
- **Exact evidence:** route `rates-inventory.controller.ts:108`; service `crs-engine.service.ts:290-444`; `tx.reservations.update` `:411`; legacy counter writes `:391-402`; **zero in-repo callers** (web uses only `:91 quote` and `POST /reservations`); E-5 = `UNKNOWN` — "no gateway container, no nginx, no ops logs in repo" (`06_EXECUTION_EVIDENCE.md:100,2107`).
- **Current behavior:** live, unaudited cross-domain write; mutates `reservations` and legacy counters in one transaction and never touches `RESERVATION_AVAILABILITY_PORT`.
- **Problem:** every use desynchronises authority from the reservation row (H-1 generator). This is B-1's "removal of `crs-engine` inventory calls" verbatim.
- **Options:**
  - **A. Retire the route** once E-5 shows no external consumer.
  - **B. Keep + deprecate on a published date** with a replacement contract.
  - **C. Keep + harden** (route consults the assertion port and fails closed).
- **Consequences:** A ends the divergence immediately; B preserves unknown external contracts behind a bounded window; C removes the divergence without breaking consumers but adds a path that must later be removed anyway.
- **Recommendation:** **obtain E-5 first**, then A if empty, otherwise B with C as the interim if the window is long. C must never be the end state — FDS §24.4 requires the legacy writer to cease when the last CRS booking/modify path is re-pointed (BR-5-018/025).
- **Final decision:** `EVIDENCE GAP` (E-5) → `REQUIRES OWNER DECISION` on the date; the disposition itself is `RESOLVED BY EXISTING DOMAIN RULE` (retire).
- **Authority:** Product owner (external contract) + engineering.
- **Ratified rule:** BR-5-017/018/025, FDS §21.4 schedule column, FDS §24.4.
- **Acceptance criteria:** after the decision, `rg "engine/modify"` in-repo returns zero callers or a recorded deprecation date; H-1 has no remaining generator.
- **Dependencies:** E-5; TD-6-02. **Downstream:** H-1, H-2, H-7, G-2, G-3, I-2.
- **Verification:** external-consumer inventory (gateway/ops, outside repo) plus in-repo caller scan.

### BD-6-04 — `gba.consumers.cascade` deploy-invariant timing (O-4)

- **Source finding:** O-4, J-2, N-3, M-5, X-6.
- **Exact evidence:** `.env.example:101` — "**DEPLOY INVARIANT: MUST be ON at deploy** (Phase 4 §6, FDS §28.3)"; absent from `.env` → false; `events.consumer.ts:151,172` therefore never invoke the pickup cascade; plan-sanctioned as an operational action `phase-5/04_…:1421`; ratified as P-21 (`phase-5/03_…:115`).
- **Current behavior:** documented deploy action never executed (audit J-AUD-01, N-3).
- **Problem:** the documented end-state and the actual flag census disagree.
- **Options:** **A.** execute the invariant now (config change only, no code); **B.** formally amend P-21 to a later trigger.
- **Consequences:** A restores the ratified posture and starts canonical pickup ledger writes; B changes a ratified rule and would need explicit re-ratification (prohibited here).
- **Recommendation:** **A**, as an operational release action sequenced with BD-6-05, because cascade ON begins writing the canonical ledger.
- **Final decision:** `RESOLVED BY EXISTING DOMAIN RULE` (P-21 already decides it); the outstanding item is execution, not policy.
- **Authority:** Operational release owner.
- **Acceptance criteria:** `.env` carries `FEATURE_GBA_CONSUMERS_CASCADE=true` with a recorded gate note (FDS §28.2); no flag changed inside a code commit (G-5).
- **Dependencies:** release window. **Downstream:** H-5 dual pickup ledgers, J-7 sequencing.
- **Verification:** post-flip evidence record plus observed pickup cascade.

### BD-6-05 — Canonical pickup ordering and soak definition (O-5)

- **Source finding:** O-5, J-6, J-7, X-6, B-9.
- **Exact evidence:** `.env.example:105-106` "sequencing: canonicalRead → soak → canonicalWrite"; Phase 4 operational action 3 (`17_PHASE4_READINESS_REVIEW.md` §8): "`canonicalRead` ON → T-66 soak → `canonicalWrite` ON → soak exit gate → document parity window results"; the sequence may not be altered (FDS §28.3).
- **Current behavior:** both flags OFF; reads go to legacy `allotment_pickups` (`allotment.controller.ts:346,379`); both ledgers are written only when the cascade is ON (`reservation-pickup-cascade.service.ts:102,113`).
- **Problem:** the **order** is ratified, but nothing anywhere defines what "soak" means in duration or evidence terms — so the sequence cannot be executed.
- **Options:** fixed calendar window; parity-threshold window (N consecutive reconciler runs with zero `LEGACY_DRIFT`); event-count window; or "no soak, flip both together" (violates the prohibition).
- **Consequences:** a parity-threshold definition is verifiable against the existing `LEGACY_DRIFT` detector (`gba-reconciliation.service.ts:350` vs `:357`) but requires `gba.reconciliation.enabled` ON (TD-6-11); a calendar window needs no new machinery but proves less.
- **Recommendation:** parity-threshold window on `LEGACY_DRIFT` with a minimum calendar floor — it makes the already-built reconciler the soak instrument instead of building a new one.
- **Final decision:** ordering `RESOLVED BY EXISTING DOMAIN RULE`; soak exit criteria `REQUIRES OWNER DECISION`.
- **Authority:** Product/operations owner.
- **Ratified rule:** FDS §28.3 sequencing; P-21; B-9.
- **Acceptance criteria:** a written soak exit criterion naming both a duration and an evidence threshold, referenced by the eventual flag-flip records.
- **Dependencies:** TD-6-11. **Downstream:** G-10 legacy pickup ledger removal.
- **Verification:** `gba-reconciliation` report history across the window.

### BD-6-06 — Typed reservation delete-reject (O-6)

- **Source finding:** O-6, L-7, M-1.
- **Exact evidence:** `RESERVATION_HAS_AVAILABILITY_STATE` defined only at `apps/api/src/common/exceptions/availability-error-codes.ts:9,21` — **never thrown**; `reservation.repository.ts:878-913` `delete()` performs no availability-state check; evidence `:858` itself states delete still surfaces "the exact raw-error class T5-59 will convert to typed".
- **Current behavior:** delete of a reservation with availability state fails on the DB `Restrict` FK (`schema.prisma:17399,:17426`) — protected, but as a raw error.
- **Problem:** ratified rule BR-5-043 / INV-P5-22 / AC-34 requires rejection "with a deterministic typed business error **before any mutation**". The rule exists; the code does not.
- **Options:** **A.** implement the typed reject (throws before any mutation); **B.** formally descope and accept raw FK rejection indefinitely.
- **Consequences:** A satisfies the ratified invariant and gives operators a clean 409; B records an accepted deviation against a locked spec.
- **Recommendation:** **A** — this is not a policy question; the policy is already ratified.
- **Final decision:** `RESOLVED BY EXISTING DOMAIN RULE` → `DIRECT TECHNICAL CORRECTION REQUIRED`.
- **Authority:** Engineering (no product choice remains).
- **Ratified rule:** BR-5-043; INV-P5-22 (`03_…:1149`); AC-34 (`03_…:1217`); FDS §26.4 (`:1046`); G-10 (`:1289`).
- **Acceptance criteria:** given a terminal reservation with availability state rows, delete is rejected with the typed error and **no** mutation occurs, balances/movements untouched; given a reservation without such rows, delete performs no availability operation.
- **Dependencies:** none. **Downstream:** reservations delete UX; Phase 11 retention guidance.
- **Verification:** unit test at the `reservation.repository.ts` boundary covering the AC-34 pair.

### BD-6-07 — GBA `pickupPct` client math (O-7)

- **Source finding:** O-7, L-6, G-14.
- **Exact evidence:** `GroupBookingsListView.tsx:101`; `GroupBookingDetailView.tsx:101,109,114` compute `pickupPct` client-side; permitted by `CLIENT_MATH_ALLOWED` at `apps/web/features/availability/__tests__/ws-n-scan-gates.test.ts:118-122` (allowlists: `group-allotment/views/`, `app/(dashboard)/inventory/`, `app/(dashboard)/lost-found/`); T5-36 — "GBA/ledger views: server values, no client recomputation" (`04:832`) **has no execution record anywhere** (L-3, L-6).
- **Current behavior:** client-side computation, sanctioned by an allowlist Phase 5 itself introduced.
- **Problem:** a plan task says "no client recomputation"; an executed allowlist says otherwise. Both cannot be the standing rule.
- **Options:**
  - **A. Complete T5-36** — server supplies the percentage; remove the two allowlist entries.
  - **B. Formally sanction the allowlist** — record `pickupPct` as a presentation-derived KPI outside BR-5-012's "availability math".
  - **C. Leave it unrecorded** — prohibited; it leaves the M-1 contradiction open.
- **Consequences:** A touches GBA views for a non-availability KPI; B closes M-1 honestly at no cost; C leaves a live contradiction.
- **Recommendation:** **B**, with an explicit written note that BR-5-012/AC-20 governs *availability* arithmetic (sellable/available figures) while `pickupPct = picked ÷ contracted` is a GBA block metric. If the owner instead wants a single zero-allowlist invariant, choose A.
- **Final decision:** `REQUIRES OWNER DECISION` (scope).
- **Authority:** Product owner.
- **Acceptance criteria:** exactly one reading is written down; `CLIENT_MATH_ALLOWED` is either emptied for GBA views or annotated with the ratified reason.
- **Dependencies:** TD-6-08. **Downstream:** WS-N scan gates; Phase 11 retirement-candidate register (G-14).
- **Verification:** scan-gate run showing removal or annotated persistence.

### BD-6-08 — Phase 4 wash carry-overs BLK-1 / BLK-2 (O-8)

- **Source finding:** O-8, X-5, B-8.
- **Exact evidence:** `phase-5/04_…:1410-1425` (§32, seven carry-overs Phase 5 "may not assume … may not execute"); Phase 4 §7 (`17_PHASE4_READINESS_REVIEW.md`): "Deviation A+B gate **wash**"; `gba.wash.schedulerEnabled` OFF and annotated "BLOCKED on Deviations A+B"; `wash-scheduler.service.ts:31`.
- **Current behavior:** dormant and correctly blocked.
- **Options:** implement `group_blocks.block_type` per spec §A-17.1 plus a durable wash/release/attrition store; or remove/replace the wash feature.
- **Consequences:** implementation unblocks the scheduler; removal changes GBA product behaviour.
- **Final decision:** `DEFERRED` to wash activation + `REQUIRES OWNER DECISION`.
- **Authority:** Product owner (Phase 4 carry-over; a later phase may not execute it unilaterally — L-r-24).
- **Dependencies:** B-8. **Downstream:** J-3 flag.

### BD-6-09 / BD-6-10 — Analytics contract and outbound publication (O-9, O-10)

- **Evidence:** P-20 (`03_…:114`) "Analytics computation decisions (real-time vs batch) are out of scope (backlog)"; P-19 (`03_…:113`) "External channel/CRS push contracts and multi-property contracts are out of scope (deferred)"; DS-06/DS-07 deferred with authority.
- **Final decision:** `DEFERRED` — both are explicitly outside the Phase 6 boundary fence (audit §B.3). **NO ACTION REQUIRED** in Phase 6.

---

## E. Technical Decisions

### TD-6-01 — `rtFilterRt` alias defect (O-11, N-1)

- **Source finding:** N-1.
- **Exact evidence:** `availability-sales.controller.ts:306` builds `AND UPPER(rt.room_type) = …`, interpolated at `:333` into a query whose only aliases are `rc`/`rd` (`:329-330`) → Postgres *missing FROM-clause entry for table "rt"*; the query executes **unconditionally** (`:326`), so it breaks in **both** flag states; reachable from `AvailabilityPage.tsx:785` (`roomType: roomTypeFilter || undefined`, dispatched unconditionally `:782`) and `AvailableRatesMatrix.tsx:40`; recorded as not-fixed by Phase 5 at `06_EXECUTION_EVIDENCE.md:681` and never fixed afterwards. Secondary: `rtFilter` (`:305`) is valid for query 1 (`FROM rooms r`) but invalid for query 3 (`:349`) and query 6 (`:400`) — flag-OFF only.
- **Current behavior:** SQL error whenever `roomType` is supplied.
- **Problem:** a live defect on the A3 authority surface Phase 6 must keep healthy.
- **Options:** **A.** correct the alias only; **B.** parameterise all interpolated filters/identifiers.
- **Consequences:** A is minimal but leaves the interpolation pattern that produced the defect (N-6); B is the durable fix and also discharges TD-6-10.
- **Recommendation:** **A and B as one change** — same code region; separating them doubles regression surface.
- **Final decision:** fix required = `RESOLVED BY TECHNICAL EVIDENCE`; parameterisation = `RECOMMENDED — CONFIRMATION REQUIRED`.
- **Authority:** Engineering.
- **Acceptance criteria:** `GET /availability/matrix?roomType=X` returns rows with no SQL error in **both** flag states; queries 3 and 6 correct under flag OFF; a **DB-backed** regression test exercises the real query path (the existing mocked specs `t38-authoritative-matrix.spec.ts` / `t520-matrix-projection-conformance.spec.ts` cannot catch it).
- **Dependencies:** none. **Downstream:** M-4/C-5.4 anchor re-verification (TD-6-12).
- **Verification:** one executed DB-backed test for the filtered matrix in both flag states.

### TD-6-02 — `modifyReservation`: port call vs retirement (O-12)

- **Source finding:** O-12, N-2, H-1, H-7.
- **Exact evidence:** `crs-engine.service.ts:364-369` gates on legacy `checkAvailability`; `:411` `tx.reservations.update`; `:391-402` legacy counters. Contrast `reservation.repository.ts`, which writes `replaceReservationAssertion` then `tx.reservations.update` with **no** legacy counter write, and in which `this.crs` / `this.inventoryDomain` are no longer invoked (verified during this analysis).
- **Options:** **A.** consult the assertion port inside `modifyReservation`; **B.** retire the route (BD-6-03); **C.** leave as-is.
- **Consequences:** A removes H-1/H-7 without touching external contracts but adds a second authority caller on a legacy path; B removes the divergence outright; C lets the core reconciliation problem generate drift indefinitely.
- **Recommendation:** **B** when E-5 allows; **A** only if E-5 proves the consumer must stay for a defined window (C as the transitory shape only, never the end state).
- **Final decision:** disposition `RESOLVED BY EXISTING DOMAIN RULE` (BR-5-017/018/025, §21.4, §24.4); timing `EVIDENCE GAP` (E-5).
- **Dependencies:** BD-6-03. **Downstream:** H-1, H-2, H-7, G-2/G-3 removal, I-2 necessity (§G).

### TD-6-03 — Reconciliation persistence, cadence, repair (O-13)

- **Source finding:** I-1, I-2, N-4, I-6, I-7, O-13.
- **Exact evidence:** `availability-reconciliation.service.ts:14-34` compares `availability.available` (`:18`) against `canonical.sellableAvailable` (`:22`), sets `automaticRepair: false` (`:32`), warn-logs each variance (`:25`), and has **no cron, no finding store, no controller persistence**; exposed only at `availability.controller.ts:21-28` with **no web caller** (K-17); no `@Cron` anywhere under `modules/availability` (I-7); the only scheduled reconciler in the codebase is GBA's, flag-OFF (`gba-reconciliation.service.ts:131`).
- **Ratified boundary (must not change):** REQ-25.1 sanctioned comparison; REQ-25.2 duties — flagged facts never auto-repaired, **legacy never wins**, dual-write drift monitored, balances are never "fixed"; REQ-25.3 reconciliation is evidence, may not feed availability decisions and may not write domain state (`03_…:1011-1020`).
- **Options for mechanism:** **A.** extend the existing service with a scheduled report-only job; **B.** new outbox consumer driven by an assertion-lifecycle event (I-6); **C.** leave on-demand.
- **Consequences:** A is the smallest change and reuses the sanctioned route; B gives event-driven precision but requires the missing event (TD-6-14); C leaves H-2 invisible between manual runs — inadequate as cutover evidence.
- **Recommendation:** **A** for the Phase 6 readiness gate (scheduled, report-only, findings persisted outside domain state), with **B** only if TD-6-14 resolves in favour of an event.
- **Final decision:** boundary `RESOLVED BY EXISTING DOMAIN RULE` (REQ-25.x — repair is forbidden, period); mechanism `RESOLVED BY TECHNICAL EVIDENCE` = scheduled report-only job; persistence of findings `RECOMMENDED — CONFIRMATION REQUIRED` (where findings live must demonstrably not violate REQ-25.3).
- **Authority:** Engineering.
- **Acceptance criteria:** a scheduled run exists; output is stored as **reconciliation evidence**, not domain state (FDS §26.2 third layer); `automaticRepair` remains `false`; legacy never wins in any output; no availability decision reads the output.
- **Dependencies:** TD-6-14 (for option B). **Downstream:** H-2, cutover evidence (§H), BD-6-05 soak instrument.
- **Verification:** an executed scheduled run with a persisted report and proof that no write statement executed.

### TD-6-04 — Population cutover mechanism (O-14)

- **Source finding:** O-14, I-3, B-1/B-2/B-3.
- **Exact evidence:** `reservation_availability_state.population ∈ {LEGACY, ASSERTION_MANAGED}` exists as a schema CHECK; **nothing assigns `LEGACY`** and **no migration tool exists** (I-3); D-21 forbids bulk backfill (`availability-phase3-implementation-plan.md:438`); first-touch already assigns (`availability-assertion.service.ts:271-275` — `ensureAndLockState` creates the row, then `population !== 'ASSERTION_MANAGED'` ⇒ `RESERVATION_POPULATION_MISMATCH` 409 at `:274`); 1124 reservations had no state row at Phase 3; `setPopulationInTransaction` exists on the port (`reservation-availability.port.ts:71`) and in the service (`availability-assertion.service.ts:632`).
- **Current behavior:** no-row ⇒ first touch assigns `ASSERTION_MANAGED`; explicit `LEGACY` ⇒ fail-closed 409. The guard is written but **nothing yet produces `LEGACY` rows**, so it is inert until cutover tooling exists.
- **Options:** **A.** census + first-touch only (`LEGACY` stays a future affordance); **B.** census + bounded-batch assignment of `LEGACY` to in-flight reservations not yet touched, converted on next assertion; **C.** bulk backfill (prohibited).
- **Consequences:** A is the least invasive and exactly what D-21 prescribes, but leaves the split unknown until the census exists; B makes the split explicit early and lets `RESERVATION_POPULATION_MISMATCH` become meaningful; C is out of bounds.
- **Recommendation:** **B preceded by a read-only census** — B-1 requires "deterministic population migration **at scale**" and only a census establishes the denominator. The assignment tool must be a **new, reviewed, transactional** tool that writes only `reservation_availability_state`, never balances or movements.
- **Final decision:** mechanism `RESOLVED BY EXISTING DOMAIN RULE` (D-21: first-touch, no bulk backfill); the census/assignment tooling itself `RESOLVED BY TECHNICAL EVIDENCE` as required Phase 6 work.
- **Authority:** Engineering.
- **Acceptance criteria:** a read-only census reporting counts by `population` and by "no row"; assignment (if chosen) is idempotent, hotel-scoped, transactional, writes no balance/movement rows, and is reversed only by re-assertion; `RESERVATION_POPULATION_MISMATCH` stays fail-closed; **no** balance backfill ever occurs.
- **Dependencies:** D-21; FDS §24.3 (no premature cuts). **Downstream:** §H cutover gates, I-2 reconciliation population.
- **Verification:** census report; assignment dry-run diff; proof that a `LEGACY` reservation is refused until first-touch.

### TD-6-05 — `evaluateRestrictions` strategy (O-15)

- **Source finding:** O-15, H-3, G-8.
- **Exact evidence:** `crs-engine.service.ts:133-197` with 3 live callers (`:245`, `:375`, FO integration `crs-front-desk-integration.service.ts:89`, unresolved adapter `:25`); `prisma-restriction.adapter.ts:23,24,114,116,153` excludes or fails closed on unproven stores; BR-5-046 mandates flag/exclude rather than guess.
- **Options:** consult the authority (as `consultAuthorityEligibility` at `crs-engine.service.ts:446-484` does) vs keep the raw read.
- **Recommendation:** keep the raw read **until E-1**, then convert; converting earlier would push every rate-scoped caller to `UNRESOLVED`.
- **Final decision:** `EVIDENCE GAP` (E-1) + interim `RESOLVED BY EXISTING DOMAIN RULE` (BR-5-046 fail-closed; no migration without evidence).
- **Dependencies:** E-1. **Downstream:** BD-6-02 steady state.

### TD-6-06 — Duplicate `GET /tax-rates` (O-16, N-5)

- **Exact evidence:** `availability-sales.controller.ts:155` and `banquet-refs.controller.ts:45`, both `@Controller()`, both under `api/v1` (`main.ts:21`); runtime winner recorded as `BanquetRefsController` by registration order (`06:199-208`); "No aliasing attempted"; merge owned by "T5-27/T5-90 disposition"; E-7 remains a Phase 5 open evidence item.
- **Final decision:** `DEFERRED` — non-availability hygiene. It is not an availability-truth divergence and must not be pulled into the Phase 6 core.
- **Acceptance criteria:** the recorded winner remains accurate until merged; the BR-5-044 single-owner duty stays open.

### TD-6-07 — Phase 3 register re-baselining (O-17)

- **Exact evidence:** M-3 (`availability-phase3-reservation-lifecycle-forensic-audit.md:758` says `holdInventory/confirmHold/releaseHold` are "no-op stubs … never injected"; the adapter implements them via assertions and is pinned by `reservation-hold-mechanism.postgres.spec.ts`); C-5.5 (no `phase-6` reference exists anywhere in the corpus).
- **Options:** annotate stale lines with a "verified stale on \<date\> — see `01_FORENSIC_AUDIT.md`" pointer, or rewrite the Phase 3 document.
- **Recommendation:** **annotate**. Phase 4/5 convention treats closed documents as immutable evidence; rewriting corrupts cross-phase traceability (audit §C.4 D-6/D-7).
- **Final decision:** `RESOLVED BY TECHNICAL EVIDENCE` — annotate, never rewrite; re-verify any Phase 3 register entry before consuming it.
- **Acceptance criteria:** every Phase 3 line consumed by Phase 6 carries a re-verification note or a fresh citation.

### TD-6-08 — Disposition of `T5-11`, `T5-36`, `T5-41`, `T5-59` (O-18)

Full disposition table is in **§J**. Summary: **T5-59 → implement (ratified rule)**; **T5-36 → owner scope decision (BD-6-07)**; **T5-11 → re-derive as a Phase 6 verification, not a retro-closure**; **T5-41 → covered indirectly, no new action, existing pins stand**.

### TD-6-09 — Canonical read omits assertion-balance consumption (D-AUD-01) — NEW FINDING

- **Source finding:** raised by this analysis; not present in `01_FORENSIC_AUDIT.md`.
- **Exact evidence:**
  - `ReservationConsumptionAdapter.read` excludes `ASSERTION_MANAGED` reservations — `reservation-consumption.adapter.ts:17-30`, comment: "ASSERTION_MANAGED reservations are represented **solely by assertion balances**, so this source must not count them (dual-count guard)" (C-07/X-13).
  - `availability-source.adapter.ts:140` passes that quantity through as `reservationConsumption`.
  - `domain/policies/snapshot-calculator.ts:14` — `consumption = reservationConsumption + gbaRemaining + allotmentRemaining`. **No balance term.**
  - `availability-snapshot.service.ts:88` feeds `facts.reservationConsumption ?? 0` into the calculator; `:98` emits it as the day's `reservationConsumption`.
  - `availability_assertion_balances` is read **nowhere** in the read path: repo-wide the only readers are inside `availability-assertion.service.ts` (the write path, plus `capacityRejection` at `:894-900`, where `:900` is `remaining = day.sellableAvailable − balances.get(stayDate)`).
  - **Consequence (a):** `GET /properties/:id/availability/snapshot` and `GET /availability/matrix` therefore **over-report `sellableAvailable`** for any stay containing ASSERTION_MANAGED reservations — i.e. essentially every reservation created since the authority went live (first-touch assigns on assert, `availability-assertion.service.ts:271-275`).
  - **Consequence (b):** `/reconciliation` computes `expected = canonical.sellableAvailable` (`availability-reconciliation.service.ts:22`) and marks `MATCH` only when `observed === expected` (`:24`) — so the sanctioned comparison inherits the same over-statement, and `DIAGNOSTIC_VARIANCE` readings are not trustworthy as cutover evidence until this is resolved.
  - **Non-evidence either way:** the parity soak (`t510-parity-soak.spec.ts`) compares **matrix against snapshot** (two authority surfaces; `:143` uses `fact.reservationConsumption`), never against balances, and seeds no ASSERTION_MANAGED reservations — it cannot detect this.
- **Current behavior:** the **write gate is correct** (assert-time `remaining = sellableAvailable − balances`); the **read contract is not** (no balance term); nothing in the corpus declares which is intended.
- **Decision question:** Is `reservationConsumption` in the canonical read supposed to cover **all** reservation commitment, or only the legacy population?
- **Ratified rules bearing on it:**
  - P-11 — exactly four fact kinds, and only their combination into sellable availability is single-sourced (`03_…:105`); the fourth kind is "**reservation commitment** (reservation consumption)" (`03_…:389`).
  - P-12 — Availability is the sole source of the sellable number; "no independent 'available' number anywhere" (`03_…:106`).
  - FDS §11.3 payload includes `reservationConsumption` as the reservation-commitment figure (`03_…:477`).
  - FDS §26.2 — assertion balances are **domain state**, "the current business truth" (`03_…:1032`).
  - C-07 / exactly-one-counting intent — each reservation counted by **exactly one** of {legacy source, assertion balances}.
  - Under these, excluding ASSERTION_MANAGED rows from the source is right (they are counted by balances); but the balances must then appear somewhere in the combination. At assert time they do. In the read they do not.
- **Options:**
  - **A. Correct the read** — include assertion balances in the snapshot's reservation commitment, either merged into `reservationConsumption` or as an additive provenance-labelled field, so snapshot, matrix, and `/reconciliation` all reflect it.
  - **B. Keep the read as-is** and declare the contract "post-assertion subtraction", requiring every reader to subtract balances itself — impossible for frontends (they cannot see balances) and effectively a second combination engine, violating P-11/P-12.
  - **C. Document the gap and defer** to Phase 11 — leaves Phase 6 cutover evidence (§H) measuring a biased number.
- **Consequences:** A changes a Phase-5-published read contract (an additive field is safe; changing `reservationConsumption` semantics is not) and moves parity baselines and `/reconciliation` variances upward; matrix and snapshot move together, so the existing parity relation is preserved. B is not viable. C makes the BD-6-05 soak and the §H cutover gates unmeasurable.
- **Recommendation:** **A, additive field** (e.g. `assertionBalanceConsumption`) combined by the snapshot calculator into `consumption` / `sellableAvailable`, with the legacy-population field left byte-identical so no existing consumer breaks. Because this amends a ratified read contract it must not be slipped in: it needs an explicit sign-off and, if approved, a dated amendment note against FDS §11.3 (annotate, do not rewrite — same rule as TD-6-07).
- **Final decision:** `RECOMMENDED — CONFIRMATION REQUIRED`.
- **Authority:** Engineering owner, with product acknowledgement (`sellableAvailable` is operator-visible).
- **Acceptance criteria (if approved):** for a property with N ASSERTION_MANAGED reservations and no legacy population, `snapshot.sellableAvailable = capacity − N` (today it equals `capacity`); matrix and snapshot agree under both flag states; `/reconciliation` `expected` equals the same figure; a DB-backed test seeds ASSERTION_MANAGED balances and proves the delta; no existing field's shape changes.
- **Dependencies:** none technically; sequenced **before** any cutover measurement (§H).
- **Downstream impact:** `AvailabilityPage`, `AvailableRatesMatrix`, `useAvailabilitySnapshot` consumers; `/reconciliation` severities; any soak baseline captured before the correction must be re-captured.
- **Verification:** one DB-backed test per surface (snapshot, matrix, reconciliation) with assertion-managed reservations present.

### TD-6-10 — Raw SQL identifier/filter interpolation (N-6)

- **Exact evidence:** `availability-sales.controller.ts:305-307` build `rtFilter`/`rtFilterRt`/`rcFilter` by interpolation into `$queryRawUnsafe` with quote-escaping only; `:377` and `:390` embed `roomType` directly.
- **Recommendation:** parameterise; bundle with TD-6-01 (same region). The audit's own assessment stands: injection risk is mitigated by `replace(/'/g,"''")`, but identifier interpolation is what produces N-1.
- **Final decision:** `RECOMMENDED — CONFIRMATION REQUIRED`.

### TD-6-11 — Activation of `gba.reconciliation.enabled` and `gba.pickup.twoLayerConsult` (J-4, J-5)

- **Exact evidence:** both absent from `.env` → false; FDS annotates `gba.reconciliation.enabled` "OFF until the soak sequence (FDS §28.3)"; `twoLayerConsult` OFF ⇒ pickup ignores authority (documented A-12 risk); `runReconciliation(hotelId)` on demand ignores the flag (`gba-reconciliation.service.ts:122`).
- **Final decision:** `DEFERRED`. Neither has a ratified activation order. If BD-6-05 adopts the parity-threshold soak, `gba.reconciliation.enabled` becomes a **precondition** of that soak — a sequencing observation, not a new rule.

### TD-6-12 — Stale anchors and stale prose (M-4, C-5.4, D-AUD-02)

- **M-4:** Phase 5 evidence anchors the matrix defect at `availability-sales.controller.ts:296-311`; current anchors are `:305-307` and `:333`.
- **D-AUD-02 (new):** FDS §24.1 states the reservation modify path writes **both** the assertion engine (`replaceReservationAssertion`, `reservation.repository.ts:621`) **and** the legacy counters (`crs.modifyReservation`, `:625`) "in one transaction … drift by design (F-04)". Post T5-50 the `:625` leg is gone: `reservation.repository.ts` writes `replaceReservationAssertion` then `tx.reservations.update` with **no** legacy counter write, and `this.crs` / `this.inventoryDomain` are never invoked. The remaining legacy writer is the **external** route only (M-8, consistent).
- **D-AUD-03 (new):** FDS §26.2 lists assertion balances as the domain-state truth, while the read path never reads them (TD-6-09) — the specification and the implementation disagree about where the fourth fact kind is materialised.
- **Final decision:** `RESOLVED BY TECHNICAL EVIDENCE` — at Phase 6 execution, re-verify every `path:line` anchor consumed; annotate FDS §24.1 as partially stale (the "dual-write by design" clause now applies only to the external `engine/modify` path, not the repository path). **Do not rewrite** closed Phase 4/5 documents.
- **Downstream:** any Phase 6 document quoting `:625` must quote M-8 instead.

### TD-6-13 — Allowlist / dead-DI removal timing (G-14, G-15, G-16)

- **Final decision:** `DEFERRED`. Each removal rides with the retirement it accompanies (G-14 entries as sites retire; G-15 until engine routes retire; G-16 after F-5). Removing them early breaks the scan gates Phase 5 uses as its retirement-candidate register.

### TD-6-14 — Assertion-lifecycle outbox event (I-6)

- **Exact evidence:** Phase 3 audit M-10 records "New outbox event(s) for assertion lifecycle (reconciliation/Phase 6)" as **MISSING**; no publisher exists.
- **Decision question:** does Phase 6 reconcile by scheduled poll (TD-6-03 option A) or by event (option B)?
- **Final decision:** `UNRESOLVED — REQUIRES EXPLICIT DECISION` (engineering owner). Recommended to resolve it inside TD-6-03 rather than as a standalone build.

### TD-6-15 — Phase 6 document set, stage model, task-ID space (X-7)

- **Exact evidence:** audit §C.4 D-4/D-5 — "No rule in either precedent yields a unique Phase 6 list; asserting one would be invention"; D-6 forbids inheriting `T5-nn` / `BLK-P5-nn` / `E-n` / `S3R-nnn`.
- **Final decision:** `UNRESOLVED — REQUIRES EXPLICIT DECISION`, **deliberately not decided here** (the brief prohibits inventing document trees or stage workflows). This document and `01_FORENSIC_AUDIT.md` are the only Phase 6 artifacts that exist; whether more follow is a phase-owner decision.

---

## F. Entry Blocker Matrix

Every blocker from `01_FORENSIC_AUDIT.md` §P.2, with disposition. A blocker is *dispositioned* — not necessarily *closed* — by this document.

| Ref | Blocker | Class | Disposition | Status after this document | Decision ID |
|---|---|---|---|---|---|
| **X-1** | E-5 external consumer inventory UNKNOWN | technical-unknown | Cannot be answered in-repo. Obtain gateway/ops inventory outside the repository. Blocks the BD-6-03 date and TD-6-02 timing only — **dispositions are already fixed** by BR-5-017/§21.4. | `EVIDENCE GAP` — OPEN | BD-6-03, TD-6-02 |
| **X-2** | M-1 evidence gaps (`T5-11`, `T5-36`, `T5-59`, `T5-41`) not dispositioned | process | Dispositioned individually in §J. Phase 5 remains closed; no retroactive closure is asserted. | **DISPOSITIONED** (one sub-item, BD-6-07, still awaits an owner) | TD-6-08, BD-6-07 |
| **X-3** | H-1/H-2 dual truth with no reconciler (I-2 absent) | technical-design | Two parts: (1) **stop the generator** — retire or harden the external modify path (TD-6-02, gated by X-1); (2) **build the comparator** — counter-vs-balance reconciliation (TD-6-03), bounded by REQ-25.2/25.3 (report-only). TD-6-09 must be confirmed first or the comparator measures a biased baseline. | **DISPOSITIONED** (design decided; execution not authorized) | TD-6-02, TD-6-03, TD-6-09 |
| **X-4** | H-3 `rate_restrictions` epistemic conflict (E-1 unknown writer) | technical + business | The transitional shape is already ratified (T5-14 + BR-5-046). Steady state waits on E-1, then an owner decision. | `EVIDENCE GAP` — OPEN | BD-6-02, TD-6-05 |
| **X-5** | Carry-overs BLK-1/BLK-2/Deviation C/D open | business | Deviation C (no schema change) is **already binding** for Phase 6 (P-7, L-r-23); wash deviations gate `gba.wash.schedulerEnabled` only and are `DEFERRED` with BD-6-08; cascade deploy gate noted as BD-6-04. | **PARTIALLY DISPOSITIONED** — wash sub-items OPEN | BD-6-08, BD-6-04, §O |
| **X-6** | Flag sequence J-4…J-7 unscheduled; J-2 invariant unsatisfied | operational | J-2 is already decided (P-21) — execution outstanding. J-6→soak→J-7 order is ratified (FDS §28.3) — soak criteria await BD-6-05. J-4/J-5 have no ratified order → TD-6-11 `DEFERRED`. | **PARTIALLY DISPOSITIONED** — J-4/J-5 OPEN | BD-6-04, BD-6-05, TD-6-11 |
| **X-7** | No Phase 6 document set, stage model, or task ID space | process | Explicitly **not** decided here (the brief prohibits inventing document trees). | `UNRESOLVED — REQUIRES EXPLICIT DECISION` — OPEN | TD-6-15 |

**Net:** X-2 and X-3 are dispositioned by this document; X-1, X-4, X-7 remain open on evidence or owner grounds; X-5 and X-6 are partially dispositioned. **No blocker can be closed by this document alone**, and no blocker requires a code change in order to be *dispositioned*.

---

## G. Reconciliation Decisions

Scope note: "reconciliation" here means **dual-source reconciliation closure** (B-1/B-2/B-3), not GBA wash reconciliation.

### G.1 As-built inventory (audit §I) with disposition

| Ref | Capability | Status as-built | Phase 6 disposition | Decision |
|---|---|---|---|---|
| I-1 | `AvailabilityReconciliationService` — snapshot vs legacy projection, `automaticRepair:false`, on-demand, no store | LIVE | **RETAIN as the sanctioned comparator** (REQ-25.1). Add scheduling and finding persistence as reconciliation evidence only. | TD-6-03 |
| I-2 | Counter-vs-balance (`availability.available` vs `availability_assertion_balances.asserted_quantity`) | **ABSENT** | **BUILD** — this is the literal "dual-source reconciliation closure" of B-1/B-2/B-3. Report-only. | TD-6-03 |
| I-3 | Population census / cutover tooling | **ABSENT** | **BUILD** read-only census first; assignment tool second, constrained by D-21. | TD-6-04 |
| I-4 | GBA reconciliation detectors (6, incl. `LEGACY_DRIFT`) | LIVE, flag OFF | **RETAIN**; candidate soak instrument for BD-6-05. | TD-6-11, BD-6-05 |
| I-5 | Wash scheduler | LIVE, flag OFF, blocked by A+B | `DEFERRED` (BD-6-08). | BD-6-08 |
| I-6 | Assertion-lifecycle outbox event | **ABSENT** | `UNRESOLVED — REQUIRES EXPLICIT DECISION`. | TD-6-14 |
| I-7 | Scheduled/repair job for availability | **ABSENT** (no `@Cron` under `modules/availability`) | **SCHEDULED REPORT-ONLY job**; repair explicitly forbidden. | TD-6-03 |

### G.2 Invariants governing every reconciliation decision (ratified — non-negotiable)

1. **Report, never repair.** REQ-25.2.1 and REQ-25.2.4 (`03_…:1015,1018`): discrepancies are flagged facts; balances are never "fixed" — that would violate the append-only movement schema.
2. **Legacy never wins.** REQ-25.2.2 (`:1016`): where authority and legacy differ, authority is the domain truth.
3. **Reconciliation is evidence, not authority.** REQ-25.3 (`:1020`): it may not feed availability decisions, eligibility, or publication, and may not write domain state.
4. **Only one sanctioned comparison.** REQ-25.1 (`:1011`): `/availability/reconciliation` is the legacy-vs-authority comparison. Any counter-vs-balance comparator must either extend this route or be introduced as an additional **sanctioned** comparison with the same read-only boundary — never as a second "truth".
5. **Layer discipline.** FDS §26.2 (`:1030-1037`): reconciliation output lives in the *reconciliation evidence* layer; a report may never become a second business authority.
6. **Availability-state ownership.** FDS §26.3 (`:1041`): only the assertion engine may create/update/delete availability-state rows; a reconciler may read them and nothing else.
7. **No auto-repair under any flag.** `automaticRepair` stays `false` in every configuration and every future job.

### G.3 Decisions taken

| Question | Decision | Status |
|---|---|---|
| Do we repair automatically? | **No** — ever. | `RESOLVED BY EXISTING DOMAIN RULE` |
| Which value wins on divergence? | **Authority.** | `RESOLVED BY EXISTING DOMAIN RULE` |
| Is counter-vs-balance comparison in scope? | **Yes** — it is B-1/B-2/B-3 verbatim. | `RESOLVED BY EXISTING DOMAIN RULE` (scope) |
| What mechanism delivers it? | Scheduled report-only job extending I-1; event-driven option contingent on TD-6-14. | `RESOLVED BY TECHNICAL EVIDENCE` |
| Where do findings persist? | As reconciliation evidence outside domain state (FDS §26.2 layer 3), hotel-scoped. | `RECOMMENDED — CONFIRMATION REQUIRED` (REQ-25.3 boundary must be demonstrable) |
| Is the current comparator sufficient for cutover evidence? | **Not until TD-6-09 is confirmed** — `expected` inherits the read omission. | `UNRESOLVED — REQUIRES EXPLICIT DECISION` before §H measurement |
| What cadence? | Hourly-adjacent, matching the GBA precedent (`gba-reconciliation.service.ts:131`), behind a flag default OFF (P-21). | `RECOMMENDED — CONFIRMATION REQUIRED` |

---

## H. Cutover Decisions

### H.1 Ratified ordering (preserved verbatim)

From FDS §24.2 (`03_…:957-971`):

> **DS-01 operational → DS-03 endpoint dispositions → DS-05 write-leg removal.** … No step may be skipped.

| Step | Precondition (ratified) | Phase 6 relevance |
|---|---|---|
| DS-02 execution (frontend cutover to authority numbers) | DS-01 operational (E-1 + E-3 + evaluator) | E-1 remains OPEN → **cannot start** |
| DS-03 read retirement (BR-5-015/016) | may proceed in parallel (read side) | largely done (P-4: engine routes 404) |
| DS-03 booking retirement (BR-5-017/019) | operational authority | the `engine/modify` retirement = BD-6-03 |
| DS-05 leg removal (BR-5-025) | DS-01 **and** DS-03 dispositions landed | repository leg already removed (T5-50); external leg pending BD-6-03 |
| `gba.a3.authoritative` ON (BR-5-037) | evaluator + E-1 + E-3 + parity soak | **already ON** (`.env:68`) — see J-AUD-01 posture note |
| DS-11 part 1 wiring (BR-5-040/041) | DS-04 authority write path exists (BLK-P5-02) | — |

Plus FDS §24.3 **no premature cuts** (BR-5-026): while the authority is not operational — no partial cuts, no "temporary" removal of a legacy leg, no flag-ON experiments with the stub bound (`:973-975`).

### H.2 Cutover decisions

| ID | Question | Decision | Status |
|---|---|---|---|
| CO-6-01 | May any legacy writer be removed before E-1 closes? | **No.** §24.3 forbids partial cuts while the authority is not fully operational; E-1 is still open (`03_…:1241`). | `RESOLVED BY EXISTING DOMAIN RULE` |
| CO-6-02 | What is the first population cutover step? | Read-only census (denominator), then `LEGACY` assignment tooling, then first-touch conversion as reservations are touched (D-21). | `RESOLVED BY EXISTING DOMAIN RULE` (D-21) + `RESOLVED BY TECHNICAL EVIDENCE` (tooling) |
| CO-6-03 | May `RESERVATION_POPULATION_MISMATCH` be made live before the assignment tool exists? | It is **already live in code** and inert in practice (nothing assigns `LEGACY`). No change needed; it becomes meaningful exactly when CO-6-02 ships. | `NO ACTION REQUIRED` |
| CO-6-04 | What baseline does cutover measure against? | Unresolved until TD-6-09 is confirmed — `/reconciliation` `expected` is biased for ASSERTION_MANAGED populations. **Sequencing rule: confirm TD-6-09 before capturing any cutover baseline.** | `UNRESOLVED — REQUIRES EXPLICIT DECISION` |
| CO-6-05 | Order of the pickup flags | `canonicalRead` ON → soak (criteria = BD-6-05) → `canonicalWrite` ON. Never altered; never both at once. | `RESOLVED BY EXISTING DOMAIN RULE` (FDS §28.3) |
| CO-6-06 | May `gba.a3.authoritative` be flipped during Phase 6? | Only as a rollback (flag OFF restores the labelled legacy matrix, §24.7); never inside a code commit (G-5); any forward change carries gate evidence (§28.2). | `RESOLVED BY EXISTING DOMAIN RULE` |
| CO-6-07 | Is a flag flip part of this document's work? | **No** — out of scope by the brief; recorded as an operational action only (BD-6-04). | `NO ACTION REQUIRED` (nothing to do now) |

### H.3 Cutover exit gates (acceptance for the eventual cutover, not an authorization)

1. E-1 answered (store population + writers identified) — currently OPEN.
2. E-5 answered or a BD-6-03 date set — currently OPEN.
3. TD-6-09 confirmed and, if approved, implemented with DB-backed proof.
4. Counter-vs-balance comparator running report-only with persisted findings (TD-6-03).
5. Population census produced; `LEGACY` / `ASSERTION_MANAGED` / no-row counts known (TD-6-04).
6. H-1 generator eliminated (BD-6-03 / TD-6-02 executed) — no path mutates `reservations` stay fields without the assertion port.
7. TD-6-01 defect fixed, so the authority surface is measurable with a `roomType` filter.
8. BD-6-05 soak criteria written; if adopted, `gba.reconciliation.enabled` ON as the instrument.
9. All flags at their ratified states with recorded gate evidence (§28.2); no flag changed inside a commit (G-5).

---

## I. Legacy Retirement Dispositions

Confirmation of `01_FORENSIC_AUDIT.md` §G. **No item is removed by this document.**

| Ref | Artefact | Disposition | Phase 6 action | Decision |
|---|---|---|---|---|
| G-1 | legacy `availability` table | **KEEP** read-only → Phase 11 | none (P-16, §24.4) | `NO ACTION REQUIRED` |
| G-2 | `InventoryDomainService` | **REMOVE** | gated by BD-6-03 (its only live caller is `engine/modify`) | `DEFERRED` → BD-6-03 |
| G-3 | `crs.modifyReservation` + `POST engine/modify` | **DEFER (E-5) → REMOVE** | obtain E-5, set date, retire | BD-6-03 / TD-6-02 |
| G-4 | `evaluateRestrictions` | **DEFER** | keep until E-1 (TD-6-05) | `DEFERRED` |
| G-5 | `room_inventory` | **KEEP read-only *or* REMOVE — undecided** | owner choice | **BD-6-01** |
| G-6 | `channel_restrictions` | **KEEP** (excluded; no writer) | none | `NO ACTION REQUIRED` |
| G-7 | `restrictions` (CUTOFF) | **KEEP** (excluded; E-4 semantics undefined) | none | `NO ACTION REQUIRED` |
| G-8 | `rate_restrictions` | **DEFER** — E-1 gate | owner decision after E-1 | BD-6-02 / TD-6-05 |
| G-9 | `channel_availability` / `channel_availability_log` | **KEEP** (out of availability scope; must be fenced) | none | `NO ACTION REQUIRED` |
| G-10 | legacy `allotment_pickups` ledger | **KEEP → REMOVE after `canonicalWrite` ON** | execute the BD-6-05 sequence first | `DEFERRED` |
| G-11 | orphaned FO / guest-profile adapters | **DEFER** — Phase 11 disposal (Phase 5 §34 note 4 reserves them) | none | `DEFERRED` |
| G-12 | `PrismaInventoryReservationAdapter` | **KEEP** (F-03/F-27 pin) | none — do not delete | `NO ACTION REQUIRED` |
| G-13 | `UnresolvedRestrictionAdapter` | **KEEP** (T5-09 rollback artifact, `availability.module.ts:50`) | none — do not delete until the rollback window closes | `NO ACTION REQUIRED` |
| G-14 | `CLIENT_MATH_ALLOWED` / WS-N allowlists | **REMOVE entries as each site retires** | tied to BD-6-07 | `DEFERRED` |
| G-15 | retired-route delay allowlist (`page.tsx:140`, `use-crs-book.ts:91`, `front-office.api.ts:193`) | **DEFER** until engine routes retire | after BD-6-03 | `DEFERRED` |
| G-16 | dead `this.crs` / `inventoryDomain` DI in the repository | **REMOVE** after F-5 retires | hygiene only | `DEFERRED` |

**Additional legacy items confirmed out of Phase 6:** deletion of the legacy `availability` table (P-16 → Phase 11); admin/mobile availability surfaces (P-19/M-7, K-18/K-19 = 0 files).

---

## J. Phase 5 Closure Dispositions

Phase 5 stays **CLOSED** (12/12 exit gates, `06_EXECUTION_EVIDENCE.md:2088-2092`). The following dispositions answer "what happens next"; none of them asserts a retroactive closure.

| Task | Plan definition | Evidence state | Disposition | Status |
|---|---|---|---|---|
| **T5-11** | "Interim gate honesty test" (`04:415`) | **Never appears in the evidence document** (L-3); no `## T5-11` heading, no mention anywhere | **RE-DERIVE, do not retro-close.** The honest-gate property is separately evidenced by T5-08 (22 tests incl. conflict/absence/failure), T5-09 binding swap, and the fail-closed family. Decide whether a dedicated interim-gate test is still wanted or the existing family subsumes it. | `UNRESOLVED — REQUIRES EXPLICIT DECISION` (engineering lead); **not** a retro-closure |
| **T5-36** | "GBA/ledger views: server values, no client recomputation" (`04:832`) | **Never appears** (L-3); target client math still live at `GroupBookingsListView.tsx:101`, `GroupBookingDetailView.tsx:101,109,114`, permitted by `CLIENT_MATH_ALLOWED` | **Owner scope decision** — complete it, or formally sanction the allowlist. | **BD-6-07** `REQUIRES OWNER DECISION` |
| **T5-41** | legacy read-only pin / "no new readers" | No dedicated heading; only "T5-83 re-derives" (`:1229`); `t557-reconciliation-readonly.spec.ts` exists and is the operative pin; M-6 records the sanctioned reconciliation exception | **COVERED INDIRECTLY — no new action.** The pin exists and is green; M-6 already documents the one sanctioned reader. Re-verify at Phase 6 execution only. | `NO ACTION REQUIRED` (re-verify only) |
| **T5-59** | "Delete-reject-when-availability-state" (BR-5-043, AC-34) | **NOT IMPLEMENTED** — `RESERVATION_HAS_AVAILABILITY_STATE` defined at `availability-error-codes.ts:9,21`, never thrown; `reservation.repository.ts:878-913` has no check; evidence `:858` states the raw error still surfaces | **IMPLEMENT.** The rule is ratified (BR-5-043 / INV-P5-22 / AC-34 / §26.4 / G-10); only the code is missing. Schedule in Phase 6 or as a targeted fix, then record. | `RESOLVED BY EXISTING DOMAIN RULE` → **`DIRECT TECHNICAL CORRECTION REQUIRED`** (BD-6-06) |

**Contradiction M-1 resolution.** The Phase 5 closing claim "every task closed with recorded evidence" is **not** restated as true. Corrected statement of record: *Phase 5 closed with 12/12 exit gates PASS; four tasks (`T5-11`, `T5-36`, `T5-41`, `T5-59`) lack dedicated evidence — of which one (`T5-59`) is an unimplemented ratified rule, one (`T5-36`) awaits a scope decision, one (`T5-11`) awaits a re-derivation decision, and one (`T5-41`) is covered indirectly.* This is recorded as a **documentation correction**, not as a reopening of Phase 5.

---

## K. Requiring Business Confirmation

The complete list of items this document will not ratify. Each needs exactly one owner decision.

| # | ID | Decision question | Options | Recommendation | Blocking evidence |
|---|---|---|---|---|---|
| 1 | BD-6-01 | Remove `room_inventory`, or keep it read-only to Phase 11? | A keep / B remove | A | G-5, B-7, T5-44 |
| 2 | BD-6-02 | After E-1: does `rate_restrictions` remain a hard quote blocker, get migrated into the authority, or get permanently excluded? | A migrate / B exclude / C keep split | A if the writer is proven, else B | H-3, G-8, E-1 |
| 3 | BD-6-03 | On what date does external `/rates/engine/modify` stop? | A retire now / B deprecate with date / C harden then retire | obtain E-5 → A or B | E-5 UNKNOWN, N-2 |
| 4 | BD-6-05 | What does "soak" mean — duration and evidence threshold? | calendar / parity-threshold / event-count | parity-threshold + floor | `.env.example:105-106`, Phase 4 §8 |
| 5 | BD-6-07 | Complete T5-36, or permanently sanction the GBA client-math allowlist? | A complete / B sanction | B | L-6, G-14 |
| 6 | BD-6-08 | Implement BLK-1/BLK-2, or remove the wash feature? | A implement / B remove | owner call (Phase 4 carry-over) | B-8, Phase 4 §7 |
| 7 | TD-6-09 | Correct the canonical read to include assertion-balance consumption? | A correct additively / B declare post-assertion contract / C defer | A | D-AUD-01 evidence in §E |
| 8 | TD-6-10 | Parameterise the A3 raw-SQL filters now, or alias-fix only? | A alias only / B parameterise | B, bundled with TD-6-01 | N-6 |

**Explicitly NOT business decisions** — this document refuses to escalate them: whether the `rtFilterRt` defect is fixed (N-1 — it is); whether delete rejects typed (BR-5-043 already says yes); whether reconciliation may auto-repair (it may not); whether bulk backfill is permitted (D-21 says no); whether `room_inventory` may be used as an availability source (BR-5-045 says no).

---

## L. Resolved by Existing Domain Rule

These are **closed**. Phase 6 preserves them; no Phase 6 document may redefine them.

| ID | Rule | Citation |
|---|---|---|
| L-r-01 | A1/Availability is the sole source of the sellable number; no independent "available" number anywhere | P-12; BR-5-008; INV-18 (`03_…:106`) |
| L-r-02 | Exactly four fact kinds; only their combination into sellable availability is single-sourced, and only by Availability | P-11 (`03_…:105`, `:389`, `:394`) |
| L-r-03 | Legacy `availability` counters and GBA legacy tables: retained, read-only, never authoritative; deletion = Phase 11 | P-16 (`03_…:110`) |
| L-r-04 | Old and new availability computations do not coexist; derived/relabelled views only | P-17 (`03_…:111`) |
| L-r-05 | Flags default OFF; `gba.consumers.cascade` MUST be ON at deploy; wash blocked by Deviations A+B | P-21 (`03_…:115`), `.env.example:101` |
| L-r-06 | Outbound channel push and analytics computation are out of scope | P-19, P-20 (`03_…:113-114`), DS-06/DS-07 |
| L-r-07 | `/availability/reconciliation` is the only sanctioned legacy-vs-authority comparison; read-only | REQ-25.1 (`03_…:1011`), BR-5-014 |
| L-r-08 | Discrepancies are flagged facts, never auto-repaired; balances are never "fixed" | REQ-25.2.1/.4 (`:1015`, `:1018`) |
| L-r-09 | Legacy never wins on divergence | REQ-25.2.2 (`:1016`), TR-15.5 |
| L-r-10 | Reconciliation is evidence, not authority; may not write domain state | REQ-25.3 (`:1020`) |
| L-r-11 | Dual-write ordering: DS-01 → DS-03 → DS-05; no step skipped | FDS §24.2 (`:957-971`), S3R-G5 |
| L-r-12 | No premature cuts while the authority is not operational | BR-5-026, FDS §24.3 (`:973-975`) |
| L-r-13 | Legacy `availability` writer ceases only when the last CRS booking/modify path is re-pointed | BR-5-018/025, FDS §24.4 (`:981`) |
| L-r-14 | Deterministic write identity on every availability-bearing write | BR-5-027, FDS §24.5 (`:986-988`) |
| L-r-15 | Rollback = flag OFF or one binding line; no schema change; table deletion out of scope | FDS §24.7 (`:996-999`), Phase 5 plan §28 |
| L-r-16 | No flag flip inside a code commit; flag changes carry gate evidence | G-5, FDS §24.8.4, §28.2 |
| L-r-17 | Unproven stores are flagged/excluded, never guessed | BR-5-046, AC-40 (`03_…:1324`) |
| L-r-18 | Conflict between restriction stores ⇒ `UNRESOLVED` (no invented precedence) | BR-5-003, E-2 (`03_…:1323`) |
| L-r-19 | `room_inventory` is not an availability source; its `availableRooms` figure may not be displayed as availability | BR-5-045/046, INV-P5-31 |
| L-r-20 | Delete is terminal-only, performs no availability operation, and rejects typed when availability state exists | BR-5-042/043, INV-P5-22, AC-34, FDS §26.4 |
| L-r-21 | Movements append-only; balances = Σ movements; availability-state rows owned exclusively by the assertion engine | INV-P5-23, FDS §26.3 (`:1041`) |
| L-r-22 | Population strategy: first-touch assign, sticky, **no bulk backfill**; `LEGACY` refuses assertion fail-closed | D-21 (`…plan.md:438`), `availability-assertion.service.ts:271-275` |
| L-r-23 | No schema change, no new migration authoring in this phase | Deviation C, P-7, audit G-1 |
| L-r-24 | Phase 4 carry-overs may not be executed unilaterally by a later phase | B-8 (`04:1410-1425`) |
| L-r-25 | Closed phase documents are annotated, never rewritten; `T5-nn` IDs are Phase 5-scoped | audit §C.4 D-6, TD-6-07 |

**Transitional states already ratified, and therefore also closed for now:** T5-14's composite quote contract (`available = eligibility === 'ELIGIBLE' && !blocked` — authority for availability, legacy for restriction display) `06_EXECUTION_EVIDENCE.md:1321`; `gba.a3.authoritative` ON with labelled legacy rollback (J-1, FDS §24.7).

---

## M. Resolved by Technical Evidence

| ID | Finding | Evidence-forced resolution |
|---|---|---|
| M-r-01 | N-1 `rtFilterRt` | The alias is wrong (`:306` vs aliases `rc`/`rd` at `:329-330`) and the query runs unconditionally (`:326`). Fix is mandatory — **TD-6-01**. |
| M-r-02 | N-6 interpolation | Identifier interpolation is the defect mechanism; parameterise — **TD-6-10**. |
| M-r-03 | M-2 "read-only" overstated | True only within the repository path; the external route still writes. Correct statement: in-repo legacy writers from booking/release paths = ∅ (M-8); the sole legacy writer is `POST /rates/engine/modify`. Recorded, not "fixed" — feeds BD-6-03. |
| M-r-04 | M-4 / C-5.4 stale anchors | Current anchors `:305-307`, `:333`; re-verify at execution — **TD-6-12**. |
| M-r-05 | D-AUD-02 FDS §24.1 stale | The `:625` dual-write leg was removed by T5-50; annotate §24.1 — **TD-6-12**. |
| M-r-06 | M-3 Phase 3 register stale | Annotate and re-verify, never rewrite — **TD-6-07**. |
| M-r-07 | I-1 / I-7 no cron, no store | The mechanism must be added; the boundary is unchanged — **TD-6-03**. |
| M-r-08 | I-3 no cutover tooling | Required Phase 6 build under D-21 constraints — **TD-6-04**. |
| M-r-09 | L-7 T5-59 unimplemented | `RESERVATION_HAS_AVAILABILITY_STATE` defined but never thrown; `delete()` has no check — implement (ratified) — **BD-6-06**. |
| M-r-10 | J-2 / N-3 cascade invariant | Flag absent from `.env` while `.env.example:101` declares it a deploy invariant; execute — **BD-6-04**. |
| M-r-11 | L-5 tasks absent from §29 | `T5-11, T5-27, T5-36, T5-41, T5-44, T5-59` absent from the execution order; `T5-44` does have a heading (`:1981`). Feeds TD-6-08. |
| M-r-12 | D-AUD-01 read omission | The balance table is read nowhere in the read path (verified repo-wide); the write gate subtracts balances at `availability-assertion.service.ts:900`. See **TD-6-09**. |
| M-r-13 | D-AUD-03 spec/impl disagreement | FDS §26.2 names assertion balances as domain-state truth; the read path never materialises them. Resolved by TD-6-09 + TD-6-12 annotation. |

---

## N. Deferred

| ID | Item | Deferred to | Authority |
|---|---|---|---|
| N-r-01 | BLK-1 (GUARANTEED_BLOCK wash exclusion) | Wash activation | Phase 4 carry-over, B-8 |
| N-r-02 | BLK-2 (durable wash/release/attrition store) | Wash activation | Phase 4 carry-over, B-8 |
| N-r-03 | Availability analytics / canonical occupancy contract | Backlog | P-20, DS-06 |
| N-r-04 | Outbound channel/CRS push publication | Backlog | P-19, DS-07 |
| N-r-05 | `/tax-rates` route merge | Hygiene pass | BR-5-044, E-7 |
| N-r-06 | `gba.reconciliation.enabled` / `gba.pickup.twoLayerConsult` activation | BD-6-05 soak definition | FDS §28.3 |
| N-r-07 | WS-N allowlist entries, retired-route delay allowlist, dead repository DI | The retirement each accompanies | G-14 / G-15 / G-16 |
| N-r-08 | Legacy `availability` table deletion | Phase 11 | P-16, P-7 |
| N-r-09 | Orphaned FO / guest-profile adapters | Phase 11 | Phase 5 §34 note 4, G-11 |
| N-r-10 | `channel_restrictions` / CUTOFF `restrictions` inclusion in the authority | Until a writer exists / E-4 answered | BR-5-046, E-4 |
| N-r-11 | Phase 6 document set + stage model + task-ID space | Phase owner decision | audit §C.4 D-4/D-5, TD-6-15 |
| N-r-12 | Assertion-lifecycle outbox event | TD-6-03 mechanism choice | Phase 3 M-10, TD-6-14 |

---

## O. No Action Required

Verified correct, sanctioned as-is, or fenced out of Phase 6 — **do not re-do** (audit §P.1):

| Ref | Item | Why no action |
|---|---|---|
| O-r-01 | Re-auditing or reopening Phase 5 | CLOSED, 12/12 gates (`06:2088-2092`); P-1 |
| O-r-02 | Building the assertion engine, snapshot, restriction evaluator, restriction write, reconciliation service | All present and bound (audit §D); P-2 |
| O-r-03 | Re-pointing Reservations / Front Office / holds to the authority | Done (K-1…K-3; `availability.module.ts:45`); P-3 |
| O-r-04 | Re-retiring `engine/availability`, `engine/restrictions`, `engine/book`, `engine/release`, `rates/availability` | 404 by construction, pinned by `t522/523/524/525`; P-4 |
| O-r-05 | Removing the repository's legacy modify leg (T5-50) | Authority-only today (verified: `replaceReservationAssertion` + `tx.reservations.update`, no counter write); P-5 |
| O-r-06 | Deleting the four duplicate top-level allotment/group-block pages (T5-43) | Done (`06:1948`); P-6 |
| O-r-07 | Authoring schema or migrations | Deviation C / D-21 / G-1; P-7 |
| O-r-08 | Building admin/mobile availability surfaces | P-19/M-7 greenfield; K-18/K-19 = 0 files; P-8 |
| O-r-09 | Any GitHub fetch/pull/restore | Not performed; local tree is authority; P-9 |
| O-r-10 | `T5-41` dedicated evidence | Covered indirectly by `t557-reconciliation-readonly.spec.ts` plus the M-6 sanctioned exception; re-verify only (§J) |
| O-r-11 | Making `RESERVATION_POPULATION_MISMATCH` "live" | Already live in code and inert by design until cutover tooling exists; CO-6-03 |
| O-r-12 | `channel_availability` / `channel_availability_log` | Out of availability scope (G-9); fence only |
| O-r-13 | BD-6-09 / BD-6-10 (analytics, channels) | Explicitly outside the boundary fence (§B.3) |
| O-r-14 | GBA wash reconciliation (I-4) and wash scheduler (I-5) as availability work | Flag-OFF and GBA-scoped; not availability dual-source reconciliation |

---

## P. Required Rules and Invariants to Preserve

Non-negotiable list carried into any future Phase 6 plan. Full citations in §L; summarised here as the checkable set.

**Authority and truth**
1. One sellable number, produced only by Availability (L-r-01, L-r-02).
2. Four fact kinds only; provenance and `UNRESOLVED` always surfaced (P-11, P-13).
3. No derived or relabelled second computation (P-17).
4. Legacy never wins a divergence (L-r-09).

**Reconciliation**
5. Report-only; `automaticRepair` always `false`; balances never fixed (L-r-08, L-r-21).
6. Reconciliation output never feeds availability decisions, eligibility, or publication (L-r-10).
7. Reconciliation output lives in the FDS §26.2 *reconciliation evidence* layer only.

**Cutover and writes**
8. DS-01 → DS-03 → DS-05, no step skipped; no premature cuts (L-r-11, L-r-12).
9. The legacy counter writer ceases only after the last CRS path is re-pointed (L-r-13).
10. First-touch population strategy; no bulk backfill; `LEGACY` fail-closed (L-r-22).
11. Deterministic write identity; no silent `'system'` or `randomUUID()` identities (L-r-14).
12. Availability-state rows written only through the port (L-r-21).

**Flags and rollback**
13. Flags default OFF; no flag changed inside a code commit (L-r-05, L-r-16).
14. `canonicalRead` → soak → `canonicalWrite`; the order is never altered (CO-6-05).
15. Rollback = flag OFF or one binding line; no schema change required (L-r-15).

**Scope discipline**
16. No schema, no migration, no backfill (L-r-23).
17. No new business rule ratified without an owner (brief authority split D).
18. Closed phase documents annotated, never rewritten (L-r-25).
19. Phase 6 does not inherit `T5-nn` / `BLK-P5-nn` / `E-n` / `S3R-nnn` IDs (audit §C.4 D-6).
20. Ratified restriction semantics: unproven stores flagged/excluded; conflicts ⇒ `UNRESOLVED` (L-r-17, L-r-18).

---

## Q. Acceptance Criteria

### Q.1 Acceptance for this document (the deliverable)

- [x] Every finding ID from `01_FORENSIC_AUDIT.md` (X-1…X-7, O-1…O-18, H-1…H-7, I-1…I-7, J-1…J-7 + J-AUD-01, G-1…G-16, L-1…L-9, M-1…M-8, N-1…N-6, P-1…P-9, C-5.1…C-5.5, B-AUD-01) appears with an explicit disposition.
- [x] Every disposition uses only the permitted vocabulary (§0.1).
- [x] No business rule ratified; every genuine policy choice is `REQUIRES OWNER DECISION` (§K).
- [x] No code, schema, migration, test, flag, reconciliation run, cutover, or legacy deletion performed.
- [x] `UNKNOWN` / `EVIDENCE INSUFFICIENT` used instead of guessing (E-1, E-5, BD-6-03, TD-6-05).
- [x] Findings raised during this analysis are labelled `D-AUD-nn` and traced to evidence (TD-6-09, TD-6-12).
- [x] Phase 5 remains closed; its evidence gaps are dispositioned, not reopened (§J).

### Q.2 Acceptance for the decisions (for whoever executes Phase 6 later)

1. **BD-6-06 / T5-59:** given a terminal reservation with availability state rows, delete returns a deterministic typed business error before any mutation; balances and movement rows untouched; a reservation without such rows deletes with no availability operation.
2. **TD-6-01:** `GET /availability/matrix?roomType=X` succeeds against real data in **both** flag states; queries 3 and 6 correct under flag OFF.
3. **TD-6-09 (if approved):** a property whose reservations are entirely ASSERTION_MANAGED reports `sellableAvailable = capacity − Σ balances` on snapshot, matrix, and `/reconciliation`; existing field shapes unchanged; the parity relation between snapshot and matrix preserved.
4. **TD-6-03:** a scheduled run exists; findings persist as reconciliation evidence; no write statement executes; output never reaches an availability decision.
5. **TD-6-04:** a census report is produced; the assignment tool (if built) writes only `reservation_availability_state`, is idempotent and hotel-scoped, and never creates balances or movements; a `LEGACY` reservation is refused with `RESERVATION_POPULATION_MISMATCH` until first-touch.
6. **BD-6-03 / TD-6-02:** no in-repo path mutates `reservations` stay fields without `RESERVATION_AVAILABILITY_PORT`; `rg engine/modify` returns either zero callers or a recorded deprecation date.
7. **BD-6-04:** `FEATURE_GBA_CONSUMERS_CASCADE=true` recorded with a gate note, changed outside any commit.
8. **BD-6-05:** a written soak exit criterion naming both a duration and an evidence threshold; flag flips recorded separately.
9. **All:** the tracked-file git baseline is unchanged during this decision phase; any later execution re-verifies every `path:line` anchor (TD-6-12).

---

## R. Downstream Implications

| Downstream artifact / owner | Implication of these decisions |
|---|---|
| **Phase 6 Implementation Plan** (not created here) | Scope is now enumerated: TD-6-01, TD-6-03, TD-6-04, BD-6-06 (T5-59), TD-6-09 (if approved), TD-6-10, plus execution of BD-6-04 and BD-6-05 once defined. **Nothing else may enter the plan without a decision ID.** |
| **Reservations rebuild (Phase 9)** | BD-6-06 changes the delete contract they may rely on; TD-6-09 changes the `sellableAvailable` they may consume. Both need notice. |
| **Front Office** | Unchanged — already on the authority port (K-2); H-7 divergence disappears only after BD-6-03. |
| **GBA / Allotment** | BD-6-05 and BD-6-04 gate the pickup ledger; G-10 removal follows `canonicalWrite`; BD-6-08 gates the wash scheduler. |
| **Channels** | Out of scope (BD-6-10 deferred); must remain fenced (G-9). |
| **Phase 11 (Legacy Migration & Removal)** | Receives G-1 and G-11, whichever of G-5 the owner leaves as KEEP, and the `LEGACY` population rows if TD-6-04 option B is built. |
| **Phase 2b (DB integrity hardening)** | Unaffected — no schema change is authorized (L-r-23). |
| **Testing / evidence policy** | TD-6-01 and TD-6-09 both require **DB-backed** tests; the existing mocked matrix specs catch neither. Phase 5's exit battery remains the baseline. |
| **Documentation** | §J gives the corrected statement of record for M-1; §M carries the annotation duties for M-3, M-4, D-AUD-02; no closed document is rewritten (L-r-25). |
| **Flag posture** | J-AUD-01 stands: Phase 6 must start from an **as-is** flag census (only `gba.a3.authoritative` is ON), not from the documented end-state. |

---

## S. Open Questions

Carried forward verbatim, with what would close each.

| # | Question | What closes it | Decision ID |
|---|---|---|---|
| S-1 | Which external systems call `/rates/engine/*`? (**E-5 / X-1**) | Gateway, nginx, or ops logs — outside the repository | BD-6-03, TD-6-02 |
| S-2 | Who writes `rate_restrictions`, `room_inventory`, `out_of_order`, `out_of_service`, and the six A3 stores, per hotel? (**E-1**) | Per-hotel row counts + recency + writer identification | BD-6-02, TD-6-05, CO-6-01 |
| S-3 | Should the canonical read include assertion-balance consumption? | Owner confirmation of TD-6-09 | TD-6-09, CO-6-04 |
| S-4 | What is the soak exit criterion for `canonicalRead` → `canonicalWrite`? | Owner definition (duration + evidence threshold) | BD-6-05 |
| S-5 | Is `pickupPct` client math a permanent sanctioned allowlist, or unfinished T5-36? | Owner scope decision | BD-6-07 |
| S-6 | `room_inventory`: KEEP or REMOVE? | Owner decision | BD-6-01 |
| S-7 | Are `T5-11` and the four evidence gaps acceptable as "dispositioned", or is a Phase 5 erratum wanted? | Engineering lead call — **no retroactive closure is asserted either way** | TD-6-08, §J |
| S-8 | Reconcile by scheduled poll or by assertion-lifecycle event? | Engineering owner | TD-6-14, TD-6-03 |
| S-9 | What is Phase 6's document set, stage model, and task-ID space? | Phase owner — deliberately not decided here | TD-6-15 |
| S-10 | Do BLK-1/BLK-2 get built, or does wash get removed? | Product owner (Phase 4 carry-over) | BD-6-08 |
| S-11 | Business meaning of `restrictions` (`rate_code='CUTOFF'`) and `zero_sell_value`? (**E-4**) | Product/domain definition (unchanged from Phase 5 §31) | outside Phase 6 core |
| S-12 | Runtime winner of the `GET /tax-rates` collision? (**E-7**) | Runtime verification before any merge | TD-6-06 |

---

## T. Final Decision Summary

1. **Phase 5 stays closed.** Its four unevidenced tasks are dispositioned, not reopened: `T5-59` must be **implemented** (a ratified rule with no code — `BD-6-06`, `DIRECT TECHNICAL CORRECTION REQUIRED`); `T5-36` needs an **owner scope decision** (`BD-6-07`); `T5-11` should be **re-derived, not retro-closed**; `T5-41` is **covered indirectly — no action**. The claim "every task closed with recorded evidence" is corrected in the statement of record (§J).
2. **Ten business decisions remain business decisions.** `BD-6-01`…`BD-6-10` are listed with options and recommendations in §D and consolidated in §K. **None is ratified here.** Of them, two (`BD-6-09` analytics, `BD-6-10` channels) are `DEFERRED` outright by P-19/P-20 and one (`BD-6-04`) is already decided by P-21 and needs only execution.
3. **Eight technical decisions are resolvable from evidence.** `TD-6-01` (matrix alias defect — fix mandatory), `TD-6-03` (scheduled report-only reconciler; repair forbidden), `TD-6-04` (census then first-touch cutover under D-21), `TD-6-07` (annotate, never rewrite), `TD-6-12` (anchor and prose re-verification) are `RESOLVED BY TECHNICAL EVIDENCE`; `TD-6-02`, `TD-6-05` have their dispositions fixed by existing rules with only timing gated on evidence; `TD-6-14` and `TD-6-15` are explicitly `UNRESOLVED — REQUIRES EXPLICIT DECISION`.
4. **Two external evidence gaps remain open and cannot be closed from this repository:** **E-5** (external `/rates/engine/*` consumers) blocks the `BD-6-03` date and `TD-6-02` timing; **E-1** (restriction-store writers) blocks `BD-6-02` steady state, `TD-6-05`, and every cutover cut under CO-6-01. Both dispositions are already fixed regardless of the answers.
5. **The core Phase 6 problems are now fully specified without being executed:** stop the H-1 generator (TD-6-02/BD-6-03); build the I-2 counter-vs-balance comparator as report-only (TD-6-03); build the I-3 census and cutover tooling under D-21 (TD-6-04); resolve the H-3 epistemic conflict after E-1 (BD-6-02); fix the N-1 authority-surface defect (TD-6-01).
6. **A new finding was raised and is unresolved pending confirmation:** the canonical read never materialises `availability_assertion_balances`, so `snapshot.sellableAvailable`, the matrix, and `/reconciliation`'s `expected` all over-report for ASSERTION_MANAGED populations (**D-AUD-01 / TD-6-09**). The write gate is correct; the read contract is not demonstrably so. **Confirm before capturing any cutover baseline (CO-6-04).**
7. **Reconciliation is bounded forever by REQ-25.x:** report-only, never repair, legacy never wins, output never feeds a decision, findings stored outside domain state. Nothing in Phase 6 may relax this.
8. **Cutover ordering is unchanged and non-negotiable:** DS-01 → DS-03 → DS-05; `canonicalRead` → soak → `canonicalWrite`; flags default OFF; no flag changed inside a commit; no premature cuts. The only unscheduled piece is the **soak definition** (`BD-6-05`).
9. **Legacy retirement is confirmed, not re-opened:** 16 items retain their §G classifications — KEEP, KEEP-until-signal, or DEFER; one (`G-5 room_inventory`) awaits an owner; **nothing is deleted in Phase 6**, and the legacy `availability` table remains Phase 11.
10. **This document is complete and the work stops here.** No code, schema, migration, test, flag flip, reconciliation run, cutover, legacy deletion, or Implementation Plan was created. The next artifact, if the phase owner authorizes one, must consume §C/§K as its input gate and may not introduce a decision that lacks an ID.

---

## U. Methodology Review — retained / modified / reconsidered

Required by the brief (§15). Assessment of the Phase 4 / Phase 5 working method as applied to a reconciliation-, cutover-, and retirement-centric phase.

| # | Method element | Verdict | Reason |
|---|---|---|---|
| 1 | `01_FORENSIC_AUDIT.md` as the phase-opening artifact | **RETAINED** | Both precedents open with it; Phase 6's scope had never been audited (B-AUD-01). |
| 2 | Stage-bound numbering 01→06 with 1:1 task IDs (`04` plan, `06` evidence) | **MODIFIED** | Phase 5's model assumes a *build* phase. Phase 6 is dominated by decisions gated on two external evidence gaps; a task register would be speculative before E-1/E-5 close. Keep the `NN_UPPER_SNAKE_CASE` filename convention (audit §C.3), but **do not assume** 1:1 stage numbering. |
| 3 | Preserved-decision register (P-1…P-22) carried forward verbatim | **RETAINED** | The single strongest control in the corpus; §L is its Phase 6 instance. |
| 4 | Option matrix + status vocabulary for every decision | **RETAINED** | Used here as §0.1 with the brief's classification set added. |
| 5 | `path:line` evidence citation with status + confidence | **RETAINED, tightened** | Phase 5's anchors went stale within one phase (M-4, C-5.4). Make re-verification of anchors a standing execution duty (TD-6-12), not a one-off. |
| 6 | "Every task closed with recorded evidence" as an exit claim | **RECONSIDERED** | It was not literally true (M-1). Replace with a **coverage-measured** claim: task-register count vs evidence-heading count, published in the exit report. |
| 7 | Long-lived feature-flag rollout with `FEATURE_*` env defaults OFF | **RETAINED** | Works, but needs an as-is flag census at phase entry (J-AUD-01) and explicit soak definitions (BD-6-05) — the sequence was ratified without its exit criterion. |
| 8 | Dual-write "drift by design" transitional state | **RETAINED, corrected** | Correct for the repository path only; FDS §24.1 prose needs annotation post-T5-50 (D-AUD-02). |
| 9 | Permitting an allowlist in the scan gates (`CLIENT_MATH_ALLOWED`) | **RECONSIDERED** | Allowlists silently convert a plan task into a standing rule (L-6 vs T5-36). Require every allowlist entry to carry a decision ID and an expiry condition. |
| 10 | Read-only reconciliation with `automaticRepair:false` | **RETAINED — non-negotiable** | REQ-25.x; §G.2 lists the seven invariants that follow from it. |
| 11 | Deferring schema work (Deviation C / D-21) | **RETAINED** | Phase 6 does not need new schema; the census and assignment tools write existing tables only. |
| 12 | Cross-phase ID inheritance (`T5-nn`, `BLK-P5-nn`) | **RECONSIDERED — prohibited** | Audit §C.4 D-6 already forbids it; Phase 6 needs its own ID space once TD-6-15 is decided. |
| 13 | Treat upstream registers (Phase 3 `L-n`) as consumable truth | **RECONSIDERED** | M-3/C-5.3 show them going stale; consume only after re-verification (TD-6-07). |
| 14 | Documentation drift handling: annotate, never rewrite | **RETAINED** | Applied to M-3, M-4, D-AUD-02, and the §J statement of record. |
| 15 | Stopping at decisions before planning | **RETAINED — and this phase needs it more than Phase 4/5 did** | Phase 6's blockers are evidence and authority blockers (E-1, E-5, eight owner decisions), not code blockers. Planning first would bake guesses into a task register. |

**Net:** the Phase 4/5 methodology is **suitable for Phase 6 with two modifications** — a coverage-measured exit claim (row 6) and an allowlist-visibility rule (row 9) — and one **prohibition reaffirmed**: no cross-phase task-ID inheritance (row 12). No new methodology was invented for this document.


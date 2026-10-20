# PHASE 4 — DEPENDENCY AND GATE MAP

**Phase 4 is a documentation-only planning phase. Phase 4 completion ≠ build authorization.**
Dependency mapping is **planning evidence only** — it records authoritative sources, current statuses, and what would future-build-unlock each workstream; it does NOT authorize, start, or order any implementation. The wave sequence lives in `01` §12; this document carries the formal graph and matrices.

Sources: identical to `00` §4's verified input sets. Identities verified against `phase-2 02` (32 HKD rows), `phase-1 02_FINDINGS_REGISTER.md` (37 HK rows — cross-walk in `phase-2 07 §7`), `phase-3 03` (ODI-01…12, lines 12–23), `phase-3 04` (Gates 1–6, lines 10–42), `phase-2 07 §9/§10`.

---

## 1. Blocking graph (dependencies are interlock, not calendar)

```
Decisions
├── G1 · Bundle Execution (HKD-04/06/20/21/23/30 + HKD-10/22, spec item set in 02 §7)
│       → unlocks: WS5-locked (HKD-20/F10.x/F4.7 in-scope authority); WS6-locked (HKD-21/F11.1/F11.2); WS7-locked (HKD-04/F9.1/F9.2/F9.3); WS4-locked (HKD-23/F6.1/F6.2/F6.5; HKD-30/F6.3/F6.4)
│       (label-preserved per §4.6: no "PENDING RATIFICATION" removal until recorded owner sign-off)
├── G2 · OQ-12 Sign-off (07 §9.4) [WS9 as its only gate target]
│       → unlocks: WS9 decorator plan, housekeeping:* registry defs+grants, guard-system choice, inspector-role; propagates to NO other workstream
├── Engineering decisions (ODI-01/02, ODI-03, ODI-11, ODI-12) recorded phase-3 03:12:14:22:23
│       → WS1 (ODI-01/02); WS4 (ODI-03/12/11)
├── Operations decisions (ODI-05, ODI-06, ODI-09) [hotel ops]
│       → WS7 (ODI-05); WS6 (ODI-06/09)
├── ODI-07 (hotel ops + engineering)
│       → WS3 calibration sub-scope (F5.7 holds adopt-contract)
├── ODI-08 (owner at review cluster)
│       → WS8 endpoint choice (HKD-27 closure record)
└── Maintenance-domain lifecycle owner capability in code (UNVERIFIED, ERR-B)
        → WS7 blocked (consumption target; no invented contract)

Workstreams (initial build order mirrors 01 §12; not all in this graph are sequential)
├── WS1 · Daily Generation & Identity ──── ODI-01/02
├── WS2 · Lifecycle Enforcement ───────── (no gate; EB/APPROVED-anchored)
├── WS3 · Inspections ───────────────────── ODI-07 + ODI-11; mirror-leg F5.3 (G3 ⊂ constraint, not a blocker)
│                                          ── fail-visible FIRST; independent validation precedes any mirror-leg work
├── WS4 · Staffing Cards & Reassign ────── G1 (HKD-23/30 sub-scopes); ODI-03/12/11
├── WS5 · Guest Requests ────────────────── G1 (HKD-20); room-service contract design-verification
├── WS6 · Linen ─────────────────────────── G1 (HKD-21); ODI-06/09; linen_inventory table mapping (ERR-E)
├── WS7 · Blocks Consumption ────────────── G1 (HKD-04); maintenance-domain lifecycle existence; ODI-05
├── WS8 · Dashboards & UI Truthfulness ──── ODI-08; retirement executions in later phases' own gates (HKD-28/F1.10)
├── WS9 · Authorization [HARD-BLOCKED] ──── G2 (OQ-12 sign-off); F14.5(a)–(d) determinations
└── WS10 · Stack Retirement Executions ──── its own later-phase gate per HKD-28/F1.10 (not a Phase 4 item)
```

**Non-propagation rule (mandate §4.2):** WS9's hard block SHRINKS to WS9 only; it does not gate WS1–WS8 or WS10. WS1/WS2/WS3-EB-side/WS8-counter-side/WS10-non-execution-carry proceed unaffected.

**Mirror-family rule (F7.3/F5.3/mandate §4.4):** no new mirror consumers anywhere in this map; the fail-visible approve leg (WS3) is planned to be implemented and independently validated BEFORE any mirror-leg re-termination consideration; HKD-32 removal remains outside Phase 4 (G3's four documented prerequisites; see `00` §7 and F7.4's staged set).

---

## 2. Workstream × Gate matrix

Legend: `—` = no effect; `ENTRY` = condition must be satisfied before future build of the named work; `PENDING-RAT` = only that rule's authority activates on G1 (not the whole WS).

| WS | G1 bundle | G2 OQ-12 | G3 HKD-32 | G4 reopen guards | G5 later-phase register | G6 FO co-sign |
|---|---|---|---|---|---|---|
| WS1 | — | — | — | — | — | — |
| WS2 | — | — | — | — | — (config refresh contract is a future realization rule, F4.4 itself recorded) | — |
| WS3 | — | — | ENTRY-constraint: no new consumers; no removal | — | — (F5.3 binding) | — |
| WS4 | PENDING-RAT (HKD-23 sub-scopes; HKD-30 sub-scopes) | — | — | — | — | — |
| WS5 | ENTRY (bundle HKD-20) | — | — | — | — | — |
| WS6 | ENTRY (bundle HKD-21) | — | — | — (F11.5/OQ-14 remains case-by-case UI-phase, not a WS6 gate) | — | — |
| WS7 | ENTRY (bundle HKD-04) | — | — | — | — | — |
| WS8 | — | — | — (no new consumers) | — | — | — |
| WS9 | — | **HARD BLOCK (WS9 only)** | — | — | — | — |
| WS10 | — | — (its own gate per HKD-28 is not G3; G3 is HKD-32's) | — | — | later-phase gate per HKD-28/F1.10 | — |

Cross-note (G6/ODI-04): affects the FO comparator re-scope sub-scope (F7.2 same-wave rule / `02 §3` same-wave rule), NOT the HK workstreams directly; carried as a cross-module dependency (§6 below).

## 3. Workstream × ODI matrix

| ODI | Owner(s) (recorded) | Affects WS | Role of the ODI in that WS |
|---|---|---|---|
| ODI-01 | Engineering lead | WS1 | Business-date key composition (overrides no default; recorded family else assumed) |
| ODI-02 | Engineering lead | WS1 | Task naming / merge rules (default adopted; choice remains) |
| ODI-03 | Engineering lead (owner-visible — OQ-15 closure) | WS4 | Card JSON shape (F6.3 plan-not-work-state; shape only if cards persist) |
| ODI-04 | FO domain owner (co-sign constraint) | (cross-module) sub-scope of FO comparator re-scope work | Vacant-state set definition; NOT an HK gate |
| ODI-05 | Hotel operations (owner-delegated queue) | WS7 | Block placement-authorization roles (F9.3) |
| ODI-06 | Hotel operations (with engineering) | WS6 | Par/location model alignment (F11.1; per-location / per-room top-up pending ops input) |
| ODI-07 | Hotel operations + engineering | WS3 | Scoring calibration (F5.7 currently-held; not invented here) |
| ODI-08 | Owner at review cluster; engineering detail | WS8 | Dashboard endpoint adopt-or-remove (F12.1/HKD-27 revision cluster record) |
| ODI-09 | Hotel operations | WS6 | Operational workflow parameters (round, cadence, damage stream — F11.1 contract-level for now) |
| ODI-10 | Security owner + product owner (non-delegable) | WS9 (only) | Guard system + inspector role (F14.5(c)/(d) closed by sign-off) |
| ODI-11 | Engineering lead | WS4 (+WS3 border) | reassign-vs-send-back consolidation (both remain specified per F4.6/F4.8) |
| ODI-12 | Engineering lead (compat) | WS4 (move-room F4.8)

## 4. Workstream × ratification bundle matrix (per mandate: bundle = HKD-04/06/20/21/23/30 [USER_DELEGATED] + HKD-10/22 [PROV default])

| WS | Bundle member(s) engaged | Authority behavior (per §4.6) |
|---|---|---|
| WS1 | — | Gate-free |
| WS2 | — | Gate-free |
| WS3 | — | Gate-free (ODI-07/11 are decisions, not bundle acts) |
| WS4 | HKD-23, HKD-30 | The `[DEL, HKD-23/30]`-locked rules (F6.1–F6.5 card/provisioning model) = **PENDING RATIFICATION**; EB repair layers (HK-011/012/024) proceed independently |
| WS5 | HKD-20 | All `[DEL, HKD-20]`-locked rules (F4.7/F10.1/F10.2 + HK-001 family) = **PENDING RATIFICATION** |
| WS6 | HKD-21 | The `[DEL, HKD-21]`-locked rules (F11.1/F11.2 + issue/return/damage flows) = **PENDING RATIFICATION**; `[PROV, HKD-22]` label preserved |
| WS7 | HKD-04 | `[DEL, HKD-04]`-locked rules (F9.1/F9.2 + ODI-05 contract framing) = **PENDING RATIFICATION** |
| WS8 | — | No bundle (HKD-24/25/26/27/19 outside bundle; ODI-08 is an owner decision not a bundle act) |
| WS9 | — | No bundle (its gate is G2/OQ-12 explicitly) |
| WS10 | — | No bundle (HKD-28's own-gate discipline is not a bundle item) |

## 5. ODI decision matrix — ODI-01…ODI-12 (exact recorded fields)

*Fields per `phase-3 03` table rows (lines 12–23, verbatim-question/recorded-default/owner; statuses per the authoritative record; nothing invented, nothing relabeled). The recorded default is “the decision as-persisted in the authoritative record”; where no default exists the field is `NONE RECORDED`.*

| ODI | Question (recorded) | Recorded default | Owning role | Status / evidence | WSs affected | Blocks |
|---|---|---|---|---|---|---|
| ODI-01 | Effective business-date attribution key composition for "daily" work (HKD-11 … exact key composition is Phase 3+ design, not a mandate) | Key family `(hotel_id, effective_business_date, room_number, task_type)` unicity for daily-generation work; ids system-attributed and collision-safe (HK-005) | Engineering lead | Recorded default; engineering choice of the exact key is a formal decision | WS1 | Implementation phase (not coding start) |
| ODI-02 | Departure vs stayover task naming/merge rules (HKD-14 … exact naming/merge rules are Phase 3+ design detail) | One checkout-adjacent task per DIRTY room per effective date when no matching active non-guest work exists; task-type distinguishes trigger source | Engineering lead | Recorded default; naming/merge details are an open engineering choice | WS1 | Implementation phase |
| ODI-03 | Card JSON shape if `hk_room_assignments` day-plan cards persist (HKD-30 … shape decision if cards persist) | Cards are plan data (rooms array = plan, never work-state); supervisor flag on cards | Engineering lead (owner-visible — data-contract stakes recorded in OQ-15 closure) | Recorded default; structured-vs-free-form / keep-vs-derive still open | WS4 | Implementation phase |
| ODI-04 | Comparator re-anchored vacant-state set definition (HKD-17/18 re-scope; `04 §3` same-wave rule) | Vacant-state set re-anchored to `housekeeping_status` values that mean "no guest" (DIRTY-class included as *vacant but not clean*), within the FO-owned flow | FO domain owner — FO co-sign constraint (`07 §4` HKD-06 line) | Recorded framing; exact set membership open; **co-sign pending** | Not an HK WS — FO comparator re-scope | FO comparator sub-scope |
| ODI-05 | Block placement-authorization roles (HKD-04 … operations-process question) | Maintenance domain owns placement; HK may request; authorization roles unspecified until ops answers | Hotel operations (owner-delegated queue) | OPEN — ops input pending | WS7 | WS7 implementation sub-scope |
| ODI-06 | Linen par-location model alignment (HKD-21 … OQ-10/05 closure: par-location alignment is a Phase 3 design input) | Real persistence on `linen_inventory` (server already reads `quantity_on_hand`/`par_level`/`unit_cost`); per-location/par-room alignment TBD | Hotel operations (with engineering) | OPEN — ops input pending | WS6 | WS6 implementation |
| ODI-07 | `room_inspections` scoring semantics (HKD-27 evidence approve score hard-coded 100 / reject 0; mandate: "either specified or flagged as an open design input") | **Flagged**: spec adopts implementation semantics as the current contract (approve = pass recorded, fail = not pass) and flags score calibration as open | Hotel operations + engineering | Recorded default held (F5.7); calibration open | WS3 | Calibration sub-scope only |
| ODI-08 | `GET /housekeeping/dashboard` adoption-or-removal (HK-034; HKD-27 review-cluster record) | Server-authoritative counters required (HKD-27 normative); unused endpoint is the ready candidate **pending ratification** | Owner at review cluster (HKD-27 chapter) / engineering for detail | OPEN — owner decision pending | WS8 | WS8 endpoint choice |
| ODI-09 | Linen operational workflow details (HKD-21 closure: "persistence + audit end-to-end becomes a later implementation requirement") | Issue/return/damage flows specified at contract level; operational parameters TBD | Hotel operations | OPEN — ops input pending | WS6 | WS6 implementation details |
| ODI-10 | Guard-system choice + inspector-role determination (OQ-12: security-owner + owner explicit sign-off on the role→action matrix) | Matrix stands as engineered in `07 §9` (ratification-ready); chapter provisional | Security owner + product owner — **explicit human sign-off; not delegable** | **OQ-12 = OPEN** (`07 §9.4`); F14.5(c)/(d) open | WS9 | WS9 only (hard) |
| ODI-11 | Inspections "reassign" vs "send back to attendant" consolidation (HK-032: exists unrouted/unused candidates; HKD-15 reassignment = explicit audited act) | Both remain specified as explicit audited acts; consolidation decision flagged | Engineering lead | OPEN decision — both normative as written | WS4 (+ WS3 border) | Implementation detail only |
| ODI-12 | Scheduled room-move acceptance of READY (HK-027) | Gate tolerance per consumer recorded verbatim; one canonical re-entry rule (HKD-02/05); strictness differences documented as consumer choices | Engineering lead (compatibility assessment) | Recorded default (F2.8 holds); consumer-choice docs live | WS4 | None |

**No default invented anywhere. Where a recorded default exists it is honored verbatim; where “NONE RECORDED” the field says so.** This map is the single index; the authoritative text lives in `phase-3 03`.

## 6. Cross-module dependencies

| Dependency domain | Owning module/consumer | Status per verified record | Impact |
|---|---|---|---|
| Room-service contract (guest requests) | `room-service` module (own controller routes + persistence) | Module existence verified; the request routes verified live this session (`room-service.controller.ts:10–59`); persistence model UNVERIFIED beneath routes | WS5's integration target; no HK-owned dual lifecycle may be created |
| Maintenance OOO/OOS lifecycle | `engineering-maintenance` module | Module exists (3 files) but a bounded grep found no OOO/OOS/block lifecycle symbols (ERR-B) — owner capability **UNVERIFIED/UNBUILT in-code**. No maintenance contract invented. | WS7 hard dependency; HK's F9.1/F9.2 side must wait until an owner capability exists to consume |
| Inventory supply lifecycle (HKD-29 adjacency) | inventory/procurement domain | Record-carried: HKD-29 = inventory-spec input, NOT an HK workstream (F11.4 boundary; `phase-3 03` HKD-28/HKD-29-family rows) | WS10 supply-stack retirement stays inventory-domain-gated; HK orders only through it |
| Front Office co-sign (ODI-04/HKD-17/18 re-scope) | FO domain owner | Co-sign constraint carried per `02 §3` same-wave rule + `07 §4` HKD-06 line; not yet GIVEN anywhere in the authoritative records | FO comparator re-scope sub-scope only; no HK workstream is gate-touched by G6 |
| Reservations F-25 posture | reservations module | Record-carried: reservations writes `rooms` columns, never the mirror (F-25's rule; also check-out touchpoint verified at `check-out.handler.ts:471`) | WS3 interlock boundary; WS2/WS7 axis read-only posture protected |

## 7. Ratification & gate activation flow (who/what/sign-off evidence)

1. **Owner (product)** signs off on:
   a. the execution of the ratification bundle (G1) — evidence: the owner's ratify/veto record against each of HKD-04/06/20/21/23/30/10/22 (a Phase-2/3 record, not a Phase 4 invention);
   b. ODI-08 adopt-or-remove decision (G1-adjacent; recorded in the closure book) — evidence: endpoint decision + next-step note in the review cluster's log.
2. **Security owner + product owner** sign off on the OQ-12 matrix (G2, non-delegable) — evidence: the OQ-12 sign-off record plus explicit F14.5(a)–(d) determinations (guard choice, registry defs/grants, decorator plan, inspector role).
3. **Engineering lead** records the answered ODI set (ODI-01/02/03/11/12) — evidence: decisions appended to the ODI register rows in the direct-implementation phase, citing exact rule anchors.
4. **Hotel operations** records ODI-05/06/09 (+ ODI-07 side) — evidence: operations decision lines appended to the ODI rows.
5. **FO domain owner** co-sign on ODI-04's vacant-state set (G6).
6. **Maintenance domain owner** realizes/admits the OOO/OOS lifecycle capability in code — evidence: the decision provider's contract reaching the `engineering-maintenance` module. This is **not an owner sign-off** — it is a build+handoff dependency to realize, not a ratification add-on.

**After each gate's evidence, only the mapped work's authority activates — nothing else.** A Phase 4 exit/closure or any other gate's activation is NOT the starting gun for any other.

## 8. Downstream impact of unresolved inputs

- **Blocking a workstream's build (any WS) vs. blocking planning:** WS9 the ONLY one hard-blocked today at the sign-off level (G2). Every other WS's dependency is Sub-Scope/Feature-Blocking at the build phase (ODI answers, bundle membership, contract existence) — **Phase 4 planning continues** around them by design (§4.6 labels).
- **Downstream complexity risk:** G1 sits upstream of THREE workstreams (WS4/5/6/7-locked scopes) — a bundle delay holds pacing there; per §4.2 no cross-gate propagation is intended.
- **Sub-scope-only waits:** ODI-07 (WS3 calibration), ODI-11 (WS4/WS3 border), ODI-12 (WS4 move-room shape), OQ-14 (F1.6 case-by-case UI) — nothing else in their WSs needs them.
- **Contract-wait:** WS5 needs a real room-service persistence contract; WS6 needs to confirm the `linen_inventory` mapping (ERR-E); WS7 needs the lifecycle owner capability (ERR-B). **None of these is currently authoritatable** — the map preserves this to prevent any build based on an assumed contract.

## 9. Preservation statement

This dependency map is **planning evidence, not authorization to implement.** No gate or ODI solution is expressed as given; every status column points to the authoritative source on which the status stands and deliberately avoids asserting an untaken decision.

## 10. Statement

**Phase 4 completion ≠ build authorization.** The graphs and matrices above are structured input for a future owner-approved build phase; they do NOT start a build, authorize a test, or modify repository state.

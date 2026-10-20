# XYLO HOUSEKEEPING — PHASE 2 CROSS-DECISION CONSISTENCY AND ORDER

Document: 04_CROSS_DECISION_CONSISTENCY_AND_ORDER.md
Lifecycle step: Phase 2 — Business Rules Decision
Status: COMPLETE
Date: 2026-10-20

Purpose: verify the 32 decisions do not contradict each other or the approved owner rules, expose the dependency/circularity structure, and fix the recommended order for closing what remains open. Evidence anchors live in `03_DECISION_ANALYSIS_D01_D08.md`; statuses in `02_BUSINESS_RULES_DECISION_REGISTER.md`.

---

## 1. Approved backbone (consistency check — passes)

The five owner decisions interlock without contradiction:

| Interaction | Check | Result |
|---|---|---|
| HKD-02 (approve-only) × HKD-09/HKD-13 (ensure-missing, preserve progressed work) | If generation reset all tasks (current behavior), INSPECTED-task states would be destroyed — violating approve-only's premise. The approved ensure-missing rule protects inspected/completed tasks, so the two are mutually reinforcing. | CONSISTENT (and the check *exposes* the current code as violate-both — recorded, no implementation) |
| HKD-16 (FO owns discrepancies) × HKD-06 (front_office_status demoted) | The FO comparator currently reads `front_office_status` against HK-state values. With the column demoted, that read loses its anchor; HKD-17/18 adopt the re-scope of the vacant-state set onto `housekeeping_status` **within the FO-owned flow** (FO domain change, HK-aware). | CONSISTENT CONDITIONAL — ordering constraint recorded (§3) |
| HKD-28 (inventory owns supplies) × HKD-19 (case-by-case D-05) | No conflict: HK-facing view of supplies is a case-by-case item, separately dispositioned; the lifecycle itself has one owner. | CONSISTENT |
| HKD-19 (case-by-case) × every D-05 recommendation | No D-05 record claims blanket "required" or "retire" — each names its own evidence/queue item. | CONSISTENT |
| HKD-17 (HK never writes occupancy truth) × HKD-31 (HK read-only on `room_status` availability axis) | Same authority direction from both sides. | CONSISTENT |

## 2. Dependency graph (policy-relevant edges)

```text
Owner-policy layer (approved)             Engineering-proposed layer
─────────────────────────────             ─────────────────────────────
HKD-02 re-entry approve-only  ─┐          HKD-01 vocabulary ──┐
                              │          HKD-05 READY semantics ┤
HKD-09 ensure-missing ────────┤          HKD-11 business-date key ─┤
  └─ HKD-13 preservation ─────┤          HKD-30 assignment model ─┤
                              │          HKD-31 availability axis ┤
HKD-16 FO owns discrepancies ─┼─→  HKD-17 occupancy authority ─┤
HKD-19 case-by-case ──────────┤          HKD-06 FO column demotion ─┤
HKD-28 inventory owns supplies┘          HKD-18 lifecycle rules ─┤
                                         HKD-03 rejection loop ─┤
                    ┌────────────────────┴───────────────────┴──────┐
                    │ Shared dependency: HKD-01 (one vocabulary)     │
                    │ gates spec-writing for 02/05/06/11/14/18/24    │
                    └───────────────────────────────────────────────┘
```

Key ordering edges (complete list in §3):

- HKD-01 → HKD-02/03/05 (words must exist before transitions can be specified on them).
- HKD-02 → HKD-03, HKD-09/13 (protect inspected), HKD-24 (move-room invariant), HKD-27 (counters reflect the approved policy), HKD-31 (axis composition).
- HKD-09 → HKD-11 (the "already exists" key), HKD-13, HKD-14, HKD-15.
- HKD-16 → HKD-17 → HKD-18 → HKD-06-re-scope (ordering constraint, below).
- HKD-28 → HKD-29 (states move to the inventory domain).
- HKD-30 → HKD-11/15/24 (assignment identity, handover, move-room contract), HKD-23 (staffing single-sourcing).
- HKD-31 + HKD-06 → HKD-32 (mirror disposition family), HKD-07/08 (mirror family members).

No circular dependency was found among finalized proposals; the only cyclic *risk* is between HKD-06 (demote FO column) and HKD-18/17 (comparator re-scope), resolved by the explicit order in §3 rather than by altering either decision.

## 3. Mutually exclusive options and conflicts resolved this phase

| Conflict | Resolution recorded |
|---|---|
| "CLEAN also re-enters" vs approve-only | Owner picked approve-only (HKD-02); alternative documented, not adopted. |
| Ensure-missing vs refresh vs reset | Owner picked ensure-missing (HKD-09); refresh documented as evaluated-rejected. |
| FO stack vs restore HK stack vs retire discrepancy | Owner picked FO stack (HKD-16); HK CQRS stack retirement is disposition-only (no code action in Phase 2). |
| Inventory domain vs HK-owned supply lifecycle | Owner picked inventory (HKD-28); lifecycle detail deferred to that domain (HKD-29). |
| Case-by-case vs blanket disposition | Owner picked case-by-case (HKD-19); queue items (OQ-01/02) inherit it. |
| HKD-06 column demotion vs comparator's current read | Not a policy conflict — an execution-order constraint: Phase 3 must specify the re-anchored vacant-state set (HKD-17/HKD-18) *in the same specification wave* as the HKD-06 demotion, and Phase 4+ implements both before removing the fallback. Until then the FO comparator's behavior is unaffected by this ledger. |
| HKD-31 (HK read-only availability) vs maintenance-domain OOO/OOS ownership (HKD-04 pending) | HKD-04 placement option (a) (maintenance owns, HK/FO consume) is compatible with HKD-31; option (b) (HK writes OOO) would violate HKD-31's read-only rule for the availability axis — flagged in OQ-03 so the owner's answer cannot silently contradict the proposed axis policy. |

## 4. Recommended finalization order (for closing PROPOSED/PENDING records)

Stage 0 (**done**) — owner-policy anchors: HKD-02, HKD-09 (+13), HKD-16 (+17), HKD-19, HKD-28.

Stage 1 — vocabulary & axis lock (prerequisite for specification writing):
1. HKD-01 (owner ratify at ledger review — lightweight: words follow implemented columns).
2. HKD-31 + HKD-04 (availability axis: owner must pick the OOO/OOS owner; recommend option (a) maintenance domain).
3. HKD-05 (engineering follow-up of 01/02).

Stage 2 — lifecycle assembly:
4. HKD-03 + HKD-02 role model (with OQ-12 security ratification).
5. HKD-06 + HKD-17/18 as one wave (comparator re-scope spec'd together).
6. HKD-11, HKD-14, HKD-15, HKD-30 (generation/assignment assembly; HKD-10 owner outcome constrains handover wording).

Stage 3 — capability dispositions (owner queue):
7. OQ batch: HKD-20, HKD-21, HKD-22, HKD-23 (+HKD-12 ops inputs), HKD-10, HKD-15-confirm, HKD-27 confirm.
8. HKD-24, HKD-25, HKD-26 (small, engineering-ratifiable once 07 done).

Stage 4 — evidence closure:
9. HKD-32 census (OQ-20) → either mirror-retirement proposal for a later DB phase or indefinite-compat disposition.

Stage 1 and Stage 2 items 4–5 are the minimum Phase 3 needs for room lifecycle, status ownership, and generation/inventory-adjacent chapters; Stage 3 items block only their own chapters' spec text.

## 5. Cross-cluster checklist (mandated)

| Mandated pair | Interaction verdict |
|---|---|
| Room lifecycle ↔ status ownership | Unified via HKD-01/02/05 + HKD-31; no circularity; one vocabulary, two axes, two owners — documented coexistence, not duplication. |
| Assignment cadence ↔ assignment persistence | HKD-09/11/13 adopt per-date precedent; HKD-30 keeps tasks authoritative; consistent with approved re-entry (INSPECTED tasks preserved by construction). |
| Checkout handling ↔ room-readiness policy | HKD-14 keeps checkout as DIRTY source; HKD-02 keeps approval as the only re-entry; FO gate composition (HKD-31) closes the loop. |
| Inspection policy ↔ Front Desk availability | Reject loop (HKD-03) holds room unsaleable; READY legacy default accepted at gate (HKD-05) — documented compatibility choice, no new writes. |
| Assignment ownership ↔ shift handover | HKD-15 carry-forward with HKD-30 card/day-plan cards; shift outcome pending (HKD-10) constrains only the handover *view*. |
| Person discrepancies ↔ occupancy authority | HKD-16/17/18 single-authority chain; comparator re-scope dependency explicit (§3 row 6). |
| Supply-request ownership ↔ inventory/procurement | HKD-28/29: HK out of the lifecycle business; only an eventual HK-facing view queued (HKD-19 channel). |
| Legacy fields ↔ canonical representations | HKD-06/31 (columns) + HKD-32 (mirror) + HKD-07/08; single-source discipline stated; retirement requires census (no forced completeness). |

Nothing in the ledger requires a decision to be force-finalized for document completeness; six PENDING records and one BLOCKED record remain openly unresolved by design.

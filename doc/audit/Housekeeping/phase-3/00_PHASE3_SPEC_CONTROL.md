# XYLO HOUSEKEEPING — PHASE 3 FDS SPEC CONTROL

Document: 00_PHASE3_SPEC_CONTROL.md
Lifecycle step: Phase 3 — Final Domain Specification
Status: COMPLETE — specification authored; closure verdict recorded in `05_PHASE3_EXIT_ASSESSMENT.md` (implementation authorization remains externally gated)
Date: 2026-10-20
Author: Phase 3 FDS session (AI agent, evidence-based, per the execution mandate)
Code changes: None. Zero production, test, schema, migration, API, UI, config, or dependency modifications. Sole write location: `docs/housekeeping/phase-3/`.

---

## 1. Baseline and locked inputs (verified at session start)

- Branch `main`, HEAD `ed278bc3a6817351c5b358292109828c412f365c` (matches the recorded Phase 3 mandate reference; verified, not assumed). Working tree = pre-existing uncommitted work (891 paths) untouched; no commits made.
- Locked Phase 1 inputs (all present, never modified): `docs/audit/Housekeeping/` — 00_AUDIT_CONTROL, 01_EXECUTIVE_AUDIT_REPORT, 02_FINDINGS_REGISTER (HK-001…HK-037), 03_WORKFLOW_MAPS, 04_DOMAIN_AND_DATA_MAP, 05_VERIFICATION_AND_COVERAGE, 06_PHASE1_EXIT_ASSESSMENT.
- Locked Phase 2 inputs (all present, never modified): `docs/housekeeping/phase-2/` — 01–07 as listed in its deliverables index. Statuses, approver map, and dispositions were re-verified against the register (02 §0/§1, §2) and exit assessment (06 §1/§4) before drafting; every label used below derives from those verifications, not from the mandate text (Mandatory Correction #1).

## 2. Ratification / OQ-12 state (recorded verbatim-provenance)

- Phase 2 exit: **PARTIAL (strict)** — criterion 3 partially met by the single open OQ-12 item (`06 §2`, `06 §4`).
- OQ-12: **OPEN** pending explicit security-owner + product-owner sign-off on the role→action matrix (`07 §9.4`). The matrix is complete and ratification-ready; it does not block spec drafting, only implementation authority.
- Phase 3 entry ratification bundle (`04 §7`): owner ratify/veto of HKD-04, 06, 20, 21, 23, 30 (delegated) and HKD-10, 22 (provisional) + the OQ-12 sign-off. Gating effect: those rules enter this FDS labeled DELEGATED/PROVISIONAL; they are normative as recorded policy, not owner-approved policy, until the bundle is executed. No ratification, veto, or sign-off occurred during this session (2026-10-20); nothing in this FDS claims otherwise (Mandatory Corrections #3, #5, #7).
- HKD-32: **RETAIN** disposition stands (static + runtime census complete, `07 §3`, `07 §10`); no removal authorized; staged removal proposal with four prerequisites belongs to a later DB-phase gate.

## 3. Mandatory-corrections compliance (Mandatory Corrections §, from the mandate)

| # | Correction | Resolution |
|---|---|---|
| 1 | Existing authority takes precedence; the prompt is not itself evidence | Every label verified against `02 §0/§1`, `02 §2`, `06 §1`. The five APPROVED anchors are quoted from the register row text, not paraphrased into new policy. |
| 2 | Do not invent a Phase 3 gate; locate existing approved criteria; else document the gap | **Gap documented**: a repository-wide search for Phase 3 exit criteria across `docs/` (including `exit criteria`/`exit gate` patterns) found **no approved Phase 3 criteria set** for the Housekeeping lifecycle (Phase 1's `06` names Phase 3 as the FDS step only; `04 §7` names entry conditions, not exit criteria). The Phase 3 exit-gate criteria used by this phase are therefore adopted **from the current owner execution mandate (mandate §9)**, cited as mandate-provided and recorded below in §5. They were not self-authored by the agent. |
| 3 | Separate specification completion from implementation authorization | `05` reports criteria 1–6 (specification completion) and the external authorization state (ratification bundle + OQ-12) as distinct determinations; no unmet criterion is reclassified. |
| 4 | Resolve the HKD-23 / HK-024 scope ambiguity | **Resolved by the register itself**: HKD-23's decision text and consequence field distinguish two concerns — (i) "teams" is retired (a grouping concept with no team table, `HK-001`/`HK-024` evidence) and (ii) **attendant-provisioning write paths become a Phase 3 spec requirement** (`02 §1` HKD-23: "Staffing model single-sourced; provisioning writes become a Phase 3 requirement (HK-024)"). Retiring teams does not shrink staffing scope — attendants + sections remain the single staffing truth and their provisioning is specified (FDS Ch 6). No silent expansion/shrink. |
| 5 | Preserve the Phase 2 authority state | OQ-12 OPEN retained; HKD-32 RETAIN retained; no removal-safety claims anywhere in this FDS. |
| 6 | Protect source records; record citation errors in the errata section | Both source sets verified present and untouched; errata in §6. Source decisions were not changed. |
| 7 | Use evidence-based statuses | Labels per mandate §4 as verified (§1); status classes only, no invented approvals/owners/dates. |
| 8 | Final verdict against the adopted gate; no weakening | `05` assesses all seven adopted criteria; unmet/external items named separately; verdict follows the mandate's verdict rules verbatim. |

## 4. Authority-label scheme (mandate §4, verified against the register)

APPROVED — owner 2026-10-20: HKD-02, HKD-09, HKD-16, HKD-19, HKD-28 (+ derived HKD-13, HKD-17).
EVIDENCE_BACKED: HKD-01, 03, 05, 07, 08, 11, 14, 15, 18, 24, 25, 26, 27, 31.
RETAIN (evidence-backed basis, distinct disposition class in `02 §0`): HKD-32.
DELEGATED — ratification pending: HKD-04, 06, 20, 21, 23, 30.
PROVISIONAL — external fact pending: HKD-10, 22.
DEFERRED — accepted: HKD-12, 29.
OPEN RATIFICATION: OQ-12 / FDS Ch 14 (security & authorization).
Every normative statement in `01_FDS_SYNTHESIS.md` carries its class inline; the full mapping per rule is in `02_TRACEABILITY.md`.

## 5. Phase 3 exit-gate criteria (adopted from mandate §9; gap per Mandatory Correction #2 recorded in §3)

| # | Criterion | Acceptance |
|---|---|---|
| 1 | Every chapter written; every ledger rule carried | All normative rules traceable to a labeled HKD record; no orphan policy statements |
| 2 | Authority labels complete and honest | Per-statement class; APPROVED anchors verbatim; zero label inflation |
| 3 | Traceability closed | 32/32 HKD + 37/37 HK with per-row verdict; no "unaddressed" |
| 4 | Consistency audit passes | Ordering constraints (mandate §6) verified; cross-checks documented in `05` |
| 5 | Open inputs and external gates explicit | `03`/`04` complete; each gate's effect on Phase 4 precisely stated |
| 6 | Zero implementation | Write-set = `docs/housekeeping/phase-3/` only |
| 7 | Exit honesty | If ratification/OQ-12 sign-off has not occurred, verdict reflects it — COMPLETE (provisional-labeled) — implementation gated, not an unconditional pass |

Verdict rules (mandate §9, preserved verbatim in intent): CLOSED only if all criteria are met AND the ratification bundle + OQ-12 are recorded as resolved; otherwise the precise non-passing status with the exact outstanding human/external acts named. No criterion weakened, reclassified, or satisfied by missing evidence.

## 6. Errata and mandate-internal discrepancies (recorded; source documents never modified)

| Ref | Item | Disposition |
|---|---|---|
| ERR-01 | Mandate §3 deliverable table says the FDS contains "chapters 1–14 (below)", while mandate §5 defines chapters 1–15. | Mandate-internal numbering inconsistency. Resolved in favor of mandate §5: the FDS is authored with **15 chapters** exactly as mapped there. |
| ERR-02 | Mandate §4 label table places HKD-32 inside the EVIDENCE_BACKED row with a "(retain)" annotation; the authoritative register carries HKD-32 as a **distinct disposition row**: "RESOLVED — RETAIN (compatibility mirror; removal proposal staged for the later DB-phase gate)" (`02 §0`/§1). | The FDS uses the register's wording and row; its basis class is evidence-backed, but its disposition class is RETAIN (no removal authorization; no removal-safety claim). |
| ERR-03 | Mandate §5 Ch-5 intro labels the approve+reject block "APPROVED rules verbatim". Only the approve-only rule is an owner anchor (HKD-02, verbatim); the rejection lifecycle specifics derive from HKD-03 (**EVIDENCE_BACKED** — verified reject path). | FDS Ch 5 labels each rule per its record; no relabeling to inflate authority (Mandatory Corrections #1, #7). |
| ERR-04 | Mandate uses "role→verb" in some places and "role→action matrix" in others. | Same artifact (07 §9 matrix); FDS uses "role→action matrix" throughout. |
| ERR-05 | Mandate §5 Ch-1 calls the resolver queue item for guest requests "HK consumes/creates through it" and Ch-10 records "HK requests via FO-adjacent live routes" (register HKD-20 wording). | Register wording adopted verbatim for the rule statement; chapter text reconciles the two phrasings as one rule. |
| ERR-06 | Mandate §5 Ch-2 names `room_status_config` impacts under mirror family — register also carries HKD-07 (`dnd_status`) and HKD-08 (`room_status_config`) as separately dispositioned mirror-family members. | FDS Ch 2/7 include HKD-07/HK-08 per register; mandate's single mention is treated as shorthand, not as narrowing. |

Nothing here modifies any `02`/`07`/`06` record; all changes are Phase 3-local.

## 7. Deliverables index

| File | Deliverable |
|---|---|
| `00_PHASE3_SPEC_CONTROL.md` | This control record |
| `01_FDS_SYNTHESIS.md` | The Final Domain Specification, chapters 1–15 |
| `02_TRACEABILITY.md` | HKD↔FDS↔HK three-way matrix (32/32 + 37/37) |
| `03_OPEN_DESIGN_INPUTS.md` | Open design-input register |
| `04_EXTERNAL_GATES_AND_CONDITIONS.md` | Post-FDS gate state and Phase 4 effects |
| `05_PHASE3_EXIT_ASSESSMENT.md` | Criterion-by-criterion exit evaluation and verdict |

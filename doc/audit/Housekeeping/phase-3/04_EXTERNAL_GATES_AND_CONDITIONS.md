# XYLO HOUSEKEEPING — PHASE 3 EXTERNAL GATES AND CONDITIONS

Document: 04_EXTERNAL_GATES_AND_CONDITIONS.md
Lifecycle step: Phase 3 — Final Domain Specification
Status: COMPLETE — gate state recorded; no new gates invented beyond the Phase 2 record
Date: 2026-10-20

Effect columns state precisely what each gate changes for Phase 4 (implementation phases) and beyond. All gates are carried over from the Phase 2 record (`02 §2`, `04 §7`, `06 §4`, `07 §4`/§9/§10); none was created by this phase.

## Gate 1 — Phase 3 entry ratification bundle (owner)

- **Contents:** ratify/veto HKD-04, HKD-06, HKD-20, HKD-21, HKD-23, HKD-30 (USER_DELEGATED) and HKD-10, HKD-22 (PROVISIONAL defaults).
- **Authority:** product owner. **Current state:** not executed during this session.
- **Effect:** the affected FDS rules remain DELEGATED/PROVISIONAL-labeled (`01` inline labels; `02` Part A) — normative as recorded policy, not owner-approved policy. On ratification: labels upgrade per `00 §4` and `04 §1` (Phase 2), and the dependent-chapter consistency checks re-run. On veto: errata-driven amendment per the register's append-and-supersede procedure (`00 §6` mechanics). **No implementation authority derives from delegated/provisional text.**

## Gate 2 — OQ-12 explicit sign-off (security owner + product owner)

- **Contents:** explicit human ratification of (a) `housekeeping:*` registry definitions/grants including the inspector-role determination (manager-only vs new HOUSEKEEPING_SUPERVISOR), (b) the route decorator plan for the 62 HK routes including the guard-system choice (common `@Permissions` vs platform `@Permission`, which differ on ADMIN-bypass). Ratification-ready matrix: `07 §9`.
- **Authority:** security owner + product owner — explicit sign-off; not delegable; not submittable by repository work. **Current state:** OPEN (unchanged from Phase 2 exit).
- **Effect:** FDS Ch 14 stays **PROVISIONAL — OPEN RATIFICATION**; the access-control half of the approved HKD-02 rule cannot bind to routes until recorded. This is the *sole remaining Phase 2 gate item* (`05`/`06`). It gates **authorization implementation only** — no other chapter's implementation, and no spec drafting, is held back by it.

## Gate 3 — HKD-32 mirror-family removal execution (later DB-phase gate)

- **Contents:** RETAIN disposition stands (static + runtime census complete: `07 §3`, `07 §10`). Removal is a proposal with four prerequisites, staged for a later DB-phase gate: (1) HK approve mirror write retired/re-terminated (HKD-02(c)); (2) the FO `room_status_history` read re-scoped or re-sourced; (3) the RLS-enabled-with-zero-policies state consciously resolved; (4) divergence re-check on a populated safe copy.
- **Authority:** the owning DB phase's own approval gate. **Current state:** proposal-only; no removal authorized; no removal-safety claim.
- **Effect:** Phase 4 must not add mirror consumers (`01` F7.3 is a standing prohibition); must land the fail-visible approve leg before any mirror-leg work is considered; must not drop the mirror.

## Gate 4 — Conditional reopen guards (external operational facts)

| Guard | Triggering fact | Effect when triggered |
|---|---|---|
| HKD-10 shifts | Property evidence of HK shift systems needing dated segregation | Ch 1 scope statement ends; shift segments enter the ledger through the HKD-19 funnel; ODI-01/15-family wording adjusted |
| HKD-22 laundry | Proof of an in-house laundry operation | Separate capability study (not retroactive HK scope) |
| HKD-12/OQ-07 credits | Ops actually delivers a workload formula + scheduling-optimization requirement | Credits chapter possible via ledger amendment |
| OQ-09 headcount | Evidence of a true-headcount requirement (ops/legal) | NEW capability study; never retroactive scope |
| OQ-14 supplies view | UI-phase supplies work starts | HK-facing view decided case-by-case then |

## Gate 5 — Deferred-to-implementation register (Phase 4 effects)

The findings mapped as "specified-but-executing-later" in `02 Part B` (HK-001, 002, 004–006, 008, 009, 011, 012, 014, 016–018, 020–022, 024, 026–030) each name their owning chapter; Phase 4 planning consumes that table. Two structural notes for Phase 4, both already normative in the FDS: enforcement realizations (transition matrix, fail-loud writes, audit attribution) and retirement executions (HK stacks, dead components, string protocol) run through their own approval gates — this document changes nothing about those gates, only records them.

## Gate 6 — FO co-sign constraint (cross-module)

- The FDS Ch 7/8 same-wave linkage (HKD-06 demotion + HKD-17/18 comparator re-scope) carries the recorded FO co-sign condition (`07 §4` HKD-06 line; `04 §3`). ODI-04's exact vacant-state set belongs to the FO domain owner. **Effect:** the comparator re-scope implementation requires FO's sign-off on the re-anchored set before code, because it is not an HK-only rule.

## Standing non-gates (recorded for completeness)

- Approve-only re-entry (HKD-02) remains fully normative now — OQ-12 only binds its access-control half (Gate 2), not its policy content (`02 §2`, `01` F2.11? — see F2.11/F5.1; no decision weakens it).
- Linen/supplies/requests/blocks capability *rules* are normative spec text; their *building* waits for implementation phases — that is sequencing, not an authorization gate.
- No other external acts exist; the Phase 2 queue's remaining-open inventory is exactly Gate 2's item (`05` footer state).

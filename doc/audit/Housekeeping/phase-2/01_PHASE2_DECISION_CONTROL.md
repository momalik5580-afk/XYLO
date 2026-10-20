XYLO HOUSEKEEPING — PHASE 2 DECISION CONTROL
Field	Value
Document	01_PHASE2_DECISION_CONTROL.md
Lifecycle step	Phase 2 — Business Rules Decision
Status	COMPLETE (exit result: PARTIAL — see 06)
Date	2026-10-20
Author	Phase 2 decision session (AI agent, evidence-based analysis + recorded owner decisions)
Code changes	None. Zero production, test, schema, migration, API, UI, config, or dependency modifications. Only this phase-2 document set under docs/housekeeping/phase-2/ was created.
1. Baseline and path resolution
Phase 1 is accepted and closed. Its control record (docs/audit/Housekeeping/00_AUDIT_CONTROL.md), findings register (HK-001…HK-037), workflow maps, domain/data map, verification report, and exit assessment are the authoritative evidence base. The brief referenced doc/audit/Housekeeping/...; the repository canonical path is docs/audit/Housekeeping/... — resolved per repository state, documents used as-is, history unchanged, nothing duplicated.

Baseline document	Role in Phase 2
docs/audit/Housekeeping/00_AUDIT_CONTROL.md	Scope boundaries, evidence-label conventions (VERIFIED FACT / OBSERVED BEHAVIOR / INFERENCE / UNVERIFIED-OPEN)
docs/audit/Housekeeping/01_EXECUTIVE_AUDIT_REPORT.md	Skeleton summary; verified system shape
docs/audit/Housekeeping/02_FINDINGS_REGISTER.md	37 findings; the decision clusters below are built from them
docs/audit/Housekeeping/03_WORKFLOW_MAPS.md	Current-state workflow truth
docs/audit/Housekeeping/04_DOMAIN_AND_DATA_MAP.md	Model/field truth
docs/audit/Housekeeping/05_VERIFICATION_AND_COVERAGE.md	Static-evidence distinction (nothing described here is runtime-proven)
docs/audit/Housekeeping/06_PHASE1_EXIT_ASSESSMENT.md	Accepted PASS with recorded runtime caveat
Evidence labels are inherited; the five records below marked APPROVED additionally carry a recorded owner decision — that is a decision authority, not new runtime evidence.

2. Mission and non-goals
Phase 2 decides what the Housekeeping domain is supposed to do. Non-goals (per constraints, verified honored):

No implementation of fixes for Phase 1 findings; no schema/migration/API/UI/test/config changes.
No Phase 3 Final Domain Specification content, no Phase 4 implementation plan, no readiness review.
No reopening or expansion of Phase 1 findings; no duplication of the register.
No promotion of enhancement candidates or architecture risks into approved requirements without owner action.
No inference of "existing behavior is correct because it exists" — current behavior is evidence, never authority.
No invented industry convention posing as an XYLO rule; no invented approvals, approvers, or dates.
No deletion of anything; dispositions are recorded, execution belongs to later phases' own approval gates.
3. Decision authority matrix
Authority	Decides	Examples this phase
Product owner (interactive session, 2026-10-20)	Capability existence/retirement, module boundaries, core operating policy	HKD-02, HKD-09, HKD-16, HKD-19, HKD-28
Hotel operations (owner-delegated, queued)	Shift model, workload formula, linen/laundry scope, guest-request SLA, escalation	HKD-10, HKD-12, HKD-20–23 inputs
Engineering lead (+ owner ratification at ledger review)	Data representation, field disposition ratification, integration topology, invariants	HKD-05, HKD-06, HKD-11, HKD-14, HKD-24–27, HKD-30–32
Security/authorization owner (with owner)	HK role/permission model (built on existing registry originals)	carried in HKD-02/03 role notes; queue item OQ-12
No one (agent)	Anything not following from an approved rule + verified constraint + repo convention	enforced; all such items are PROPOSED/PENDING, never self-APPROVED
An agent recommendation is never an owner decision. Only records with an actual recorded owner response are APPROVED; two records (HKD-13, HKD-17) are APPROVED as direct decompositions of approved owner rules, with derivation stated.

4. Decision record conventions
Decision IDs: HKD-### sequential (32 records), grouped in clusters D-01…D-08 as mandated.
Statuses: PROPOSED / PENDING_OWNER_DECISION / APPROVED / REJECTED / DEFERRED / BLOCKED_BY_EVIDENCE (superset of docs/audit/Reservations/07_DECISIONS.md conventions, extended per this phase's brief; REJECTED unused so far).
Every record carries: business question, finding IDs, source evidence, current-state facts (labeled F/I/U), options, recommendation+rationale, implications, dependencies, required approver, approval status, and Phase 3 consequences.
The consolidated analysis records live in 03_DECISION_ANALYSIS_D01_D08.md (single-file per repository convention for phase artifacts, mirroring docs/reservations/phase-N/ numbered files).
5. Decision-methodology compliance
Each HKD record follows the mandated 12-step sequence (§4 of the brief): question → findings/evidence → current state → fact/inference/unknown split → operational question → viable options → trade-offs → invariants/failure cases → recommendation → authority → recorded status → dependencies. Interactive decision review (§9 protocol) was available and used: five owner questions were asked in one batch, actual responses recorded verbatim in §6, and dependent records updated for consistency.

6. Recorded owner decisions (verification transcript)
Asked 2026-10-20 (single batch, per §9: evidence + options + recommendation presented; responses recorded as given):

Q	Cluster	Question (condensed)	Owner answer	Ledger effect
Q1	D-01	Room re-entry authorization	Approve-only (Recommended) — acknowledged	HKD-02 APPROVED
Q2	D-03	Daily generation semantics	Ensure-missing (Recommended) — acknowledged	HKD-09 APPROVED; HKD-13 APPROVED (derived)
Q3	D-04	Person-discrepancy disposition	Adopt FO stack	HKD-16 APPROVED; HKD-17 APPROVED (derived)
Q4	D-06	Supply-request domain ownership	Inventory owns it (Recommended)	HKD-28 APPROVED
Q5	D-05	Default disposition policy for disconnected calls	Case-by-case (Recommended)	HKD-19 APPROVED
The "Recommended" acknowledgment is recorded as the owner's acceptance of the stated rationale, per the session record. No other approvals exist; all remaining HKD records are PROPOSED / PENDING_OWNER_DECISION / DEFERRED / BLOCKED_BY_EVIDENCE.

7. Cross-phase hygiene
Runtime-evidence rule (brief §2): no runtime probes were needed to distinguish competing policy options this phase; the latent runtime/evidence questions are carried instead as formal evidence requirements (the HKD-32 mirror census, coded OQ-20; the HKD-12 workload-formula inputs; the HKD-22 laundry operational-model evidence; the HKD-04 placement-authorization answer) — recorded, not executed. No production data touched; no test environments used.
Read-only compliance: the write-set of this session is exactly the six files in docs/housekeeping/phase-2/. Pre-existing uncommitted work preserved untouched.
Findings cross-reference discipline: original HK-### finding IDs are cited throughout; this phase creates no new forensic findings — new facts discovered during decision analysis are recorded as evidence inside HKD records (none is a new violation register entry).
8. Deliverables index
File	Deliverable
01_PHASE2_DECISION_CONTROL.md	D1 — Decision control (this file)
02_BUSINESS_RULES_DECISION_REGISTER.md	D2 — Business rules decision register
03_DECISION_ANALYSIS_D01_D08.md	D3 — Decision analysis records (consolidated)
04_CROSS_DECISION_CONSISTENCY_AND_ORDER.md	D4 — Cross-decision consistency and dependency map
05_OWNER_DECISION_QUEUE.md	D5 — Owner decision queue
06_PHASE2_EXIT_ASSESSMENT.md	D6 — Phase 2 exit assessment

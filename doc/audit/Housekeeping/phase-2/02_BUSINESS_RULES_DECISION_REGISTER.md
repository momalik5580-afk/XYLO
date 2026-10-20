# XYLO HOUSEKEEPING — PHASE 2 BUSINESS RULES DECISION REGISTER

Document: 02_BUSINESS_RULES_DECISION_REGISTER.md
Lifecycle step: Phase 2 — Business Rules Decision
Status: COMPLETE (register current as of 2026-10-20; amendment procedure: append + supersede, never rewrite history)
Date: 2026-10-20

Evidence basis: `docs/audit/Housekeeping/` (Phase 1, accepted) + targeted re-verification documented in `03_DECISION_ANALYSIS_D01_D08.md` §9. Approval statuses derive ONLY from the recorded 2026-10-20 interactive owner session (01 §6) or explicit derivation from approved rules. Agent recommendations are PROPOSED, never APPROVED.

## 0. Status summary

| Status | Count | Records |
|---|---|---|
| APPROVED (owner session 2026-10-20) | 5 | HKD-02, HKD-09, HKD-16, HKD-19, HKD-28 |
| APPROVED (derived from approved rules) | 2 | HKD-13, HKD-17 |
| PROPOSED (await ledger ratification) | 14 | HKD-01, HKD-03, HKD-05, HKD-06, HKD-11, HKD-14, HKD-15, HKD-18, HKD-24, HKD-25, HKD-26, HKD-27, HKD-30, HKD-31 |
| PENDING_OWNER_DECISION | 6 | HKD-04, HKD-10, HKD-20, HKD-21, HKD-22, HKD-23 |
| DEFERRED | 4 | HKD-07, HKD-08, HKD-12, HKD-29 |
| BLOCKED_BY_EVIDENCE | 1 | HKD-32 |
| REJECTED | — | none recorded this session |
| **Total** | **32** | |

## 1. Register

| ID | Cluster | Decision (business rule) | Status / approver / date | Phase 1 evidence | Dependencies | Phase 3 consequence |
|---|---|---|---|---|---|---|
| HKD-01 | D-01 | Authoritative room-condition vocabulary: DIRTY / CLEAN / INSPECTED (+ legacy defaults READY, task-level PICKUP); progress/combined states are task- or display-level, not room-column values | PROPOSED — owner ratification | HK-021, HK-014, HK-019; schema `rooms:11174` | HKD-02, HKD-31/32 | One vocabulary in spec; constants-file alignment is implementation-stage work |
| HKD-02 | D-01 | **Approve-only re-entry:** inspection approval is the sole authorized DIRTY→service transition; nobody else declares readiness; swallowed-failure design declared unacceptable (fix owned by later phases) | **APPROVED** — Product owner (interactive session) 2026-10-20 | HK-006, HK-016, HK-008; `room.aggregate.ts:30` | HKD-03, HKD-05, HKD-09, HKD-27; OQ-12 security ratification | Core lifecycle rule; row-mirror write must become fail-visible in spec |
| HKD-03 | D-01 | Inspection rejection leaves room DIRTY-class (unsaleable), produces re-clean task, audit-logs authenticated inspector identity | PROPOSED — engineering (+HKD-02 invariant) | HK-008, HK-006 | HKD-02; OQ-12 | Rejection loop + escalation model specified |
| HKD-04 | D-01 | OOO/OOS availability-block ownership (maintenance domain vs HK) not yet claimed by an owner module | PENDING_OWNER_DECISION — OQ-03 | HK-020, HK-019, HK-015 | HKD-31 | Block placement and authorization roles specified once owner answers |
| HKD-05 | D-01 | READY = legacy default equivalence ("accept as cleanable-ready"), never an inspection outcome/outward write target | PROPOSED — engineering (test-pinned) | HK-014; `room-readiness.spec.ts:43–44` | HKD-02 | READY/INSPECTED distinction documented; gate semantics unchanged |
| HKD-06 | D-02 | `rooms.front_office_status`: not authoritative in HK; no HK writer (zero-writer status = policy); FO comparator's HK-state read must be re-scoped (dependency Phase 3); read-fallback path designated removal | PROPOSED — engineering + owner ratification | HK-015, HK-028/029; `get-discrepancies.handler.ts:15,60` | HKD-17, HKD-18 (comparator re-scope), HKD-31 | Column demoted to compatibility; FO owns any future concept decision |
| HKD-07 | D-02 | `dnd_status` (mirror column): zero consumers; out of HK domain scope; retirement folds into mirror census | DEFERRED — engineering record | grep-verified 2026-10-20; `schema.prisma:10964` | HKD-32 | Excluded from HK spec; listed in mirror retirement evidence |
| HKD-08 | D-02 | `room_status_config` (legacy 4-state display order): no HK consumer; FO legacy | DEFERRED | `schema.prisma:10976–10988` | HKD-32 | Excluded from HK spec |
| HKD-09 | D-03 | **Ensure-missing generation:** daily generation creates missing daily work only; completed/in-progress/inspected tasks never reset; duplicates = no-op; no mid-run abort on key conflicts; trigger authority = HK supervisor (manual only; automation = separate enhancement approval) | **APPROVED** — Product owner 2026-10-20 | HK-004, HK-009, HK-005; `housekeeping.service.ts:162–207` | HKD-11, HKD-13, HKD-15, HKD-23 | Replaces destructive reset in spec; idempotency + effective-date key required |
| HKD-10 | D-03 | Shifts (day/night, schedules, property-local time): scoping unresolved — recommended OUT of HK domain scope pending ops evidence | PENDING_OWNER_DECISION — OQ-06 | HK-031 | HKD-15 (handover view), HKD-11 | Either excluded (recommendation) or first-class shift semantics |
| HKD-11 | D-03 | "Daily" requires an effective business-date attribution key (composition decided at Phase 3 design; per-date precedents exist in adjacent models) | PROPOSED — engineering, derived from HKD-09 | HK-004; `room_turndown:11023`, `room_discrepancies` uniqueness | HKD-09, HKD-13, HKD-30 | Generation idempotency formalized; HK-009 dies in spec |
| HKD-12 | D-03 | Workload/credit formulas: none exists; SLA config is display-only (no overdue engine) | DEFERRED — awaiting ops inputs | HK-036; `slaMinutes` display consumers | HKD-27 | No credits in spec until approved; SLA labeled display-only |
| HKD-13 | D-03 | Task-preservation + concurrency invariants (owner-approved decomposition of HKD-09): never reset progressed work; transactional batch; skip-on-conflict; audit reference preserved | **APPROVED (derived)** — decomposes HKD-09 | HK-004, HK-009, HK-010 | HKD-09 | Spec invariants |
| HKD-14 | D-03 | Checkout-triggered vs daily: checkout marks DIRTY (stands); generation creates checkout-adjacent work only for DIRTY rooms without matching active non-guest work; no duplicate assignment | PROPOSED — engineering (+HKD-02/HKD-09 interaction) | HK-006, `check-out.handler.ts:470–481` | HKD-02, HKD-09, HKD-11 | Checkout/daily merge rules specified |
| HKD-15 | D-03 | Handover: no automatic transfer of unfinished work; carry-forward with original assignee; reassignment = explicit audited human act | PROPOSED — engineering (+owner light confirm OQ-08) | HK-031 | HKD-09, HKD-14, HKD-30 | Carry-forward + reassignment rules specified |
| HKD-16 | D-04 | **Person discrepancy: Front Office stack is the single authority** (`room_discrepancies`); HK-tagged CQRS stack dispositionally retired; HK UI consumes FO data if/when needed | **APPROVED** — Product owner 2026-10-20 | HK-003; migration `20260829_s1`; `front-office.controller.ts:49–69` | HKD-17, HKD-18, HKD-06 | Capability defined FO-owned; HK spec references it |
| HKD-17 | D-04 | Occupancy authority: FO in-house reservations + `room_status` are the occupancy facts; HK never writes occupancy truth; comparator vacant-states re-anchored to `housekeeping_status` | **APPROVED (derived from HKD-16)** | `get-discrepancies.handler.ts:42–87` | HKD-06 (ordering), HKD-18 | Reconciliation contract specified |
| HKD-18 | D-04 | Record lifecycle: OPEN on detection w/ business-date uniqueness; RESOLVED requires user; reopen-on-recompute semantics retained (current verified behavior adopted as policy); HK supervisors consume, do not resolve | PROPOSED — engineering | `discrepancy.repository.ts:50`; `resolve-discrepancy.handler.ts:30` | HKD-16; HKD-06 (re-scope ordering) | Lifecycle & retention specified |
| HKD-19 | D-05 | **Case-by-case disposition default** for every disconnected capability; no blanket presume-required/presume-retire | **APPROVED** — Product owner 2026-10-20 | HK-001 (Annex A) | HKD-20–27 | Each group dispositioned individually in queue OQ-01/02 |
| HKD-20 | D-05 | Guest requests: room-service authoritative; HK string protocol retired in spec; HK requests via FO-adjacent live routes; persistence+audit required end-to-end | PENDING_OWNER_DECISION — OQ-04 | HK-030, HK-001 | HKD-19 | Request lifecycle rules for HK side specified after owner priority call |
| HKD-21 | D-05 | Linen management: recommended REQUIRED (real persistence; par-location alignment); scope/rounds TBD by owner | PENDING_OWNER_DECISION — OQ-10 | HK-001, HK-002, HK-035 | HKD-19, HKD-22 | Linen flows + audit specified if confirmed |
| HKD-22 | D-05 | Laundry: unresolved pending operational model (in-house vs outsourced) | PENDING_OWNER_DECISION — OQ-10 | HK-001 | HKD-19, HKD-21 | Build scope or retire-down defined |
| HKD-23 | D-05 | "Teams" concept: retire in favor of attendants+sections staffing (no teams table exists; supervisor flag already in assignment cards) | PENDING_OWNER_DECISION — OQ-11 | HK-001, HK-024 | HKD-10, HKD-30 | Staffing model single-sourced in spec |
| HKD-24 | D-05 | Task move-room: approve integration (service method exists unrouted; UI call dead); invariant: move cannot violate HKD-02 or OOO/OOS reads | PROPOSED — engineering; ledger ratification | HK-001; service method (Phase 1) | HKD-02, HKD-30 | Move-room rules specified |
| HKD-25 | D-05 | Lost-found claim affordances in HK UI: delegate to lost-found module; remove duplicates; no backend change | PROPOSED — engineering | HK-001; lost-found module pages | HKD-19 | UI disposition recorded for later UI phases |
| HKD-26 | D-05 | Six dead `_components` (incl. crash-if-render `MinibarSection`): obsolete → retirement candidates in later UI phases; no silent resurrection | PROPOSED — engineering | HK-023 | HKD-19 | Component retirement list for UI phase |
| HKD-27 | D-05 | Dashboard/statistics truthfulness: rendered operational counters must be server-authoritative; unused `GET /housekeeping/dashboard` is the ready candidate (or removed) — ratify at review | PROPOSED — engineering | HK-022, HK-034 | HKD-02 (inspected counts), HKD-12 | Counter-authority model in spec |
| HKD-28 | D-06 | **Supply requests: inventory/procurement domain owns the lifecycle;** HK orders through it; HK-tagged stack dispositionally retired (retirement executes in later phases' own gates) | **APPROVED** — Product owner 2026-10-20 | HK-037, HK-026; inventory stack glob (full module) | HKD-29, HKD-19 (HK-facing view case-by-case) | No HK supply lifecycle in spec; cross-domain view decision queued |
| HKD-29 | D-06 | Supply-request states/quantity/approval-level policies: inventory domain's concern; not an HK Phase 2 obligation | DEFERRED — by HKD-28 | HK-037 context | HKD-28 | Recorded as inventory-spec input |
| HKD-30 | D-07 | Assignment representation: per-room work ownership authoritative on `housekeeping_tasks.assigned_to`; `hk_room_assignments` = day-plan cards (supervisor flag included); card rooms-JSON is plan data, never work-state | PROPOSED — engineering; owner ratification (data-contract stakes) | HK-011; `5781–5794`, service `:214,501–557` | HKD-11, HKD-15, HKD-24, HKD-23 | Re-spec of HK-011 posture; contract-change list recorded |
| HKD-31 | D-08 | `rooms.room_status`: operational-availability axis owned by FO/reservations; HK strictly read-only; gate composes both axes (documented, not merged) | PROPOSED — engineering; owner ratification | F-25 tests; rooms defaults | HKD-02, HKD-04, HKD-32 | Axis-ownership rules specified |
| HKD-32 | D-08 | `room_status` mirror table (+history/config): compatibility mirror until census proves otherwise; mirror-writes continue only via the two approved lifecycle paths while errors stay fail-hidden → interim; **retirement BLOCKED_BY_EVIDENCE** (OQ-20 census) | BLOCKED_BY_EVIDENCE (retirement); interim PROPOSED | HK-013; `05 §7` census | HKD-02(c), HKD-06, HKD-31 | Mirror policy documented; no retirement activity authorized |

## 2. Required approver map (unfulfilled)

| Record(s) | Required approver | Vehicle |
|---|---|---|
| HKD-04, HKD-10, HKD-12, HKD-20–23 (+ confirm HKD-15) | Product owner / hotel operations | Owner Decision Queue (05) OQ-01..OQ-11 |
| HKD-01, 03, 05, 06, 11, 14, 18, 24–27, 30, 31 | Engineering lead + owner ratification at ledger review | Ledger review session |
| HKD-02/03 role model (attendant/supervisor/front-desk) | Security/authorization owner + owner | OQ-12 |
| HKD-32 census | Engineering evidence work | OQ-20 |

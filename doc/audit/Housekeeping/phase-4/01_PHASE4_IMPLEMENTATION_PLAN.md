# PHASE 4 — HOUSEKEEPING IMPLEMENTATION PLAN (WS1–WS10)

**Phase 4 is a documentation-only planning phase. Phase 4 completion ≠ build authorization.**
Every acceptance test below is a **future build-phase obligation**. None has been executed in Phase 4. No code, test, schema, migration, API, UI, or configuration artifact exists as a result of this document.

Basis: locked Phase 3 FDS (`docs/housekeeping/phase-3/` — blob-verified intact in `00` §4), Phase 2 registers (`docs/housekeeping/phase-2/`), Phase 1 findings (`docs/audit/Housekeeping/02_FINDINGS_REGISTER.md`, HK-001…HK-037).

---

## 1. Reading conventions

- Status vocabulary (§4.8 integrity): `VERIFIED (this session, read-only)` = inspected live code/document this session; `register-carried` = evidence taken from the authoritative Phase 1/2/3 record (not re-inspected this session); `UNVERIFIED` = named in a record but not yet inspected; `PENDING DECISION` = owner/ODI input outstanding; `BLOCKED` = hard gate.
- Rule citations use the **exact** Phase 3 anchor text: `[APPROVED verbatim, HKD-09]`, `[EB, HKD-05]`, `[DEL, HKD-21]`, `[RETAIN]`, `[PROV]`, `[INTENDED — OPEN RATIFICATION]`, `[EB/APPROVED-adjoined]` — never relabeled (§4.6).
- Any rule whose record sits in the **ratification bundle** (HKD-04, 06, 20, 21, 23, 30, 10, 22) carries the sub-scope label **PENDING RATIFICATION** until the bundle executes; planning proceeds around it on its recorded authority (mandate §4.6). APPROVED/EB-anchored work proceeds on its own authority.
- Every code touchpoint was inspected this session with the exact `path:line` shown; any touchpoint not verified is explicitly marked `UNVERIFIED`.
- `blocked-on` is mandatory: no workstream is planned without its full blocking-input list.

---

## 2. WS1 — Daily generation & identity

**Purpose.** Replace the destructive, id-colliding, no-key daily generation with ensure-missing generation under a system-attributed, collision-safe business-date key, exactly as the FDS specifies.

**In scope (future build):** generation rewrite for daily HK work; `ensure-missing` semantics; effective-business-date key composition (per ODI-01); checkout-adjacent task creation rules; nomenclature/merge per ODI-02; id-attribution strategy (system-allocated, collision-safe); destructive-reset removal; idempotent re-run; audit-preserved batch.

**Not in scope:** automated scheduling triggers (F3.1: "trigger authority = HK supervisor (manual only; automation = separate enhancement approval)"); shifts/day-night attribution (F1.1 [PROV], HKD-10 — out of scope); workload/SLA engine (F1.3, HKD-12; `slaMinutes` display-only); mobile surface (F13.4, HK-025).

**Verified anchors.**

| Anchor | Authority label (exact) | Bearing |
|---|---|---|
| F3.1 | `[APPROVED verbatim, HKD-09]` "Ensure-missing generation… completed/in-progress/inspected tasks never reset; duplicates = no-op; no mid-run abort on key conflicts… manual only" | Head rule |
| F3.2 | `[APPROVED-derived verbatim, HKD-13]` never reset progressed work; transactional batch; skip-on-conflict; audit reference preserved | Concurrency/preservation |
| F3.3 | `[EB, HKD-11 + ODI-01]` business-date attribution key; family `(hotel_id, effective_business_date, room_number, task_type)`; system-attributed; collision-safe; idempotent no-op | Key rule |
| F3.4 | `[EB, HK-005]` client-minted ids prohibited; `HK-${Date.now()}`/`TASK-${Date.now()}` families illegal | Identity |
| F3.5 / F3.6 / F3.7 / F3.8 | `[EB, HKD-14]` / `[EB, HK-009 resolution]` / skew / HK-004 destructive reset "detectably illegal" | Checkout-adjacent tasks; collision dead; skew bound; illegibility of reset |
| HK-004 (EB) | Generation destructively resets all daily tasks; | Sequenced as F3.8's referenced behavior |
| HK-005 (EB) | `HK-${Date.now()}`/`TASK-${Date.now()}` id families; `notif-`/`AUDIT-`-style minting | Replacement obligation |
| HK-009 (EB) | Unique-key collision aborting generation mid-loop | F3.6 dead-in-spec |
| HKD-11 (EB) | Business-date precedents `room_turndown`, `room_discrepancies` uniqueness | Only per-date, never invented before |
| ODI-01 / ODI-02 | Engineering lead; defaults recorded in `phase-3 03` — composition/naming/merge are **design decisions to be executed**, not assumed | Decision inputs |

**Verified code touchpoints (all inspected this session).**

| Touchpoint | Evidence | Role |
|---|---|---|
| `apps/api/src/modules/housekeeping/housekeeping.service.ts:162` `generateDailyTasks()` | New | Generation today (to be replaced) |
| `:165–174` `updateMany` (housekeeping_tasks; status→pending, start_time→null, completed_at→null over all non-`Guest:` rows) | New | HK-004 destructive-reset behavior (F3.8 illegal) |
| `:189` `HK-${Date.now()}-${room}-${Math.random()…}`; `:191` `TASK-${Date.now()}` | New | HK-005 collision-prone id factories |
| `:59–60` `HK-${Date.now()}` in `createTask`; `:796` `AUDIT-${Date.now()}` in `logAuditEntry`; `:653` `notif-${Date.now()}`; `:846` `SUP-${Date.now()}` | New | Multi-table id-minting families (same obligation) |
| `apps/api/src/modules/housekeeping/housekeeping.controller.ts:41` `@Post('tasks')`; `:46` `@Post('tasks/generate-daily')` | New | Trigger surface (manual only per F3.1) |
| Tables in use: `housekeeping_tasks`, `hk_room_assignments` (`:176`) | New | Persistence namespace |

**Future acceptance-test obligations.**

| Test | Maps from | Trigger/failure condition | Expected observable (from rule) | Evidence needed |
|---|---|---|---|---|
| WS1-T1 no-reset of progressed work | F3.1/F3.2, HK-004 | Generate over pending + in-progress + completed + inspected sets | Nothing in any progressed state is reset or reassigned (timestamps preserved) | Pre/post state diff + audit trail |
| WS1-T2 duplicates no-op | F3.1/F3.3 | Re-run generation same effective date, same key | No new rows, no abort | Idempotency diff |
| WS1-T3 no mid-run abort on key conflict | F3.1/F3.6, HK-009 | Force a key conflict mid-run | Conflict skipped; remaining work created; no exception leak | Seeded conflict test |
| WS1-T4 system-allocated collision-safe ids | F3.4, HK-005 | Generate with synthetic clock collisions | No `Date.now()`-shaped ids anywhere persisted | DB scan of id shapes |
| WS1-T5 checkout-adjacent linkage | F3.5, HKD-14 | Check-out (`rooms` DIRTY at `check-out.handler.ts:471`) then generate | Exactly one matching active non-guest task for the DIRTY room (per ODI-02 present in matching `_evenMoreNeverReset` matching-arrangement… not applicable — "no duplicate") | Task count per room |
| WS1-T6 skew immutability | F3.7 / F3.3 | Adversarial/clock-skew rerun | No destructive outcome; degraded to no-op or correct set | Failure-seed coverage |

**Entry / exit (future implementation).**
*Entry:* ODI-01 and ODI-02 answered by the engineering lead (recorded in the register); no other gate touches WS1 (APPROVED/EB anchors).
*Exit:* all six tests above observable; no `HK}-${Date.now()}`-family mint remains anywhere in HK module callsites (all five families on listed touchpoints); `updateMany` reset path deleted.

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| ODI-01 exact business-date key composition | Engineering lead | PENDING DECISION (default family recorded — not assumed) |
| ODI-02 task naming/merge rules | Engineering lead | PENDING DECISION (default recorded — not assumed) |

**Ratification status:** no bundle-locked sub-scopes in WS1 (all anchors APPROVED/EB); both ODI decisions are engineering work, not ratification acts.

*Planned tests have not been executed under Phase 4. Phase 4 completion ≠ build authorization.*

---

## 3. WS2 — Task lifecycle enforcement

**Purpose.** Make the specified status-transition matrix actually enforced, and make all state/config writes fail-loud.

**In scope (future build).** Enforce `F2.7`-matrix on `updateTask` (audit label map at `:152–159` already reflects the rejected paths); finalize "verification of unenforced transition matrix" (HK-026); fail-loud config writes (`hk_sla_config`, `hk_status_flow_config`) replacing `setConfig`'s catch-and-continue norm (`:903–917`, `:934` success-without-persistence); refresh contract for the per-instance config cache (`:905–906`) per F4.4; transitions audited with the acting identity (`F4.5`).

**Not in scope.** Inventing new lifecycle states or transition permissions (F2.7/F2.4 fixed vocabulary `pending | in-progress | completed | inspected | rejected | cancelled`; uppercase DISPLAY-family artifacts are display-only per F2.4); inventory/procurement transitions (HKD-29 boundary; F11.4 discipline); runtime DDL (F15.1).

**Verified anchors.**

| Anchor | Authority label | Bearing |
|---|---|---|
| F2.7 | `[EB, HK-026 + approved lifecycle HKD-02/03/09]` matrix defined; "no any-to-any"; `inspected → pending`/`cancelled` prohibited; supply-request module is the codebase's existing enforcement standard | Head rule |
| F2.4 | `[EB, HK-021]` task-status vocabulary fixed per implemented set | Vocabulary law |
| F2.1/F2.3/F2.2 | `[EB verbatim, HKD-05]`/`[EB, HKD-02/03 via 07 §7 HK-014]` | Authority of column/producer rules (consumed by WS3 too) |
| F4.2 | `[EB, F2.7 continuation; HK-026]` `pending → in-progress → completed → inspected`; `rejected` cycle per Ch 5; cancelled pre-completion only; both-directions prohibited; audited labeled | Matrix detail |
| F4.4 | `[EB, HKD-27 + HK-018]` fail-loud for config writes; success-without-persistence prohibited; uninvalidated per-instance cache "specified out" (refresh contract in implementation) | WS2 core |
| F4.5 | `[EB/APPROVED-adjoined]` authenticated identity on every mutation | Audit law |
| F15.1 | `[EB, HK-017]` no run-time DDL; raw-SQL discipline explicitly scoped; `ensureAuditLogTable`-style runtime creation is a named prohibited class | Schema guard |
| HK-026 (EB) | Transition matrix enforced nowhere today (audit map `rejected→in-progress`, `completed→pending` as "restarted"/"removed" labels at `:152–159`) | Defect record |
| HK-018 (EB) | `updateSLAConfig` returns `{success:true}` at `:934` even when `setConfig` swallowed the DB failure (`:914–916` catch → console.error → normal path) | Defect record |
| HK-021 (register-carried) | Casing/priority constants-family display mismatch | F2.4 anchor; vocabulary alignment is implementation work only (HKD-01 consequence) |
| ODI-12 | "Scheduled room-move acceptance of READY" — per-consumer gate already documented; **applies also to WS4's move-room F4.8** | Cross-reference only here |

**Verified code touchpoints.**

| Touchpoint | Role |
|---|---|
| `apps/api/src/modules/housekeeping/housekeeping.service.ts:83` `updateTask(id, data)` | Transition write path to be matrix-enforced |
| `:152–159` `mapStatusToAuditAction(from,to)` | The unenforced matrix map (labels for transitions incl. `completed→pending`, `rejected→in-progress`, `rejected→completed` — the HK-026 surface) |
| `housekeeping.controller.ts:51` `@Put('tasks/:id')`; `:56` `@Patch('tasks/:id/status')` | Route entrypoints that must reject non-matrix transitions |
| `:919–953` `getSLAConfig/updateSLAConfig/getStatusFlowConfig/updateStatusFlowConfig` | Config writers |
| `:903–917` `setConfig` (raw upsert into `property_config`, catch→continue) | HK-018 defect surface to make fail-loud |
| `housekeeping-supply-request.service.ts:8–9,44–47,64` | The module's own codebase-standard transition-guard pattern (F2.7 explicitly names this standard) |

**Future acceptance-test obligations.**

| Test | Maps from | Trigger/failure | Expected observable |
|---|---|---|---|
| WS2-T1 matrix enforcement | F2.7/F4.2 | `inspected → pending`, prohibited reverse hops, any post-completion cancel | 403/409-class rejection with a documented code; no write |
| WS2-T2 legit-path pass | F2.7/F4.2 | Each approved hop (`pending→in-progress`, `in-progress→completed`, `completed→inspected`, `rejected` cycle per Ch 5, pre-completion cancel) | Allowed; acting identity on the audit entry |
| WS2-T3 config fail-loud | F4.4, HK-018 | Force a DB error in `setConfig` | HTTP error surfaced, no `{success:true}` |
| WS2-T4 config cache invalidation | F4.4 refresh contract | Write then immediately re-read cross-instance | Newer value observed; no stale `:905–906` cache serving |
| WS2-T5 audit-label fidelity | F4.5 | Matrix-approved transitions only | `mapStatusToAuditAction` never signs an unapproved transition |

**Entry / exit (future).** *Entry:* EB anchors only; no external gate. *Exit:* prohibited transitions impossible at the route+service layers; config writes error-surfaced; no per-instance uninvalidated caches remain; all mutation paths carry authenticated identity.

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| (none) — WS2 is gate-free on EB/APPROVED anchors | — | Proceeds |

**Ratification status:** none (no bundle-locked anchors).

*Planned tests have not been executed under Phase 4. Phase 4 completion ≠ build authorization.*

---

## 4. WS3 — Inspections (approve-only, fail-visible, re-clean, spec'd gate constraints)

**Purpose.** Make the approved `HKD-02(c)` approve-only/fail-visible rule real: authenticated inspector identity, hard-coded-score replacement, rejection → re-clean production, serialize approve/reject pairs.

**In scope (future build).** Rework of `approveInspection` and `rejectInspection` in the module service (`:560/:614`) implementing exactly: F5.1 (approve-only re-entry; swallowed failures unacceptable), F5.2 (reject → re-clean auto-target; room condition never written by rejection), F5.3 (fail-visible mirror leg; the swallow family at `:577/:582/:593`), F5.4 (saleable-block while rejected/unresolved), F5.8 (serialized decisioning), F4.5 identity (client-supplied `inspectorId` `:584` prohibited), F2.3 (task-completion produces CLEAN; approval alone produces INSPECTED), F2.2 (READY never written as outcome), F5.7/ODI-07 scoring semantics as the recorded contract (hard-coded 100/0 adopt-or-recalibrate — **decision pending, not invented here**). F5.5 (no auto-escalation) and F5.6 (reject-on-occupied-room) stay respected.

**Not in scope.** Any mirror-leg *removal* (`F7.4` Stage requires HKD-32 prerequisites — never Phase 4); new mirror consumers (`F7.3`); proposing removal-safety claims (HKD-32 RETAIN); inventing score calibrations (ODI-07); consolidating "reassign vs send-back" (ODI-11, engineering decision pending — cross-cutting with WS4 too).

**Verified anchors.**

| Anchor | Authority label | Bearing |
|---|---|---|
| F5.1 `HKD-02(c)` | `[APPROVED verbatim, HKD-02]` | Sole authorized DIRTY→service transition; nobody else declares readiness |
| F5.2 | `[EB, HKD-03 + verified reject path]` | Re-clean production requirement; room-condition-no-write; no client-supplied inspector fields (HK-008 family dead) |
| F5.3 | `[APPROVED-binding + EB]` | Fail-visible mandate; swallow family `:577/582/593` prohibited |
| F5.4 / F5.5 / F5.6 / F5.7 / F5.8 | `[EB, 03 §5]` / `[EB]` / `[EB, HKD-31 interlock]` / `[OPEN, ODI-07]` / `[specified failure cases]` | Saleable-invariant + escalation owner + occupancy interlock + scoring + serialization |
| F7.3 | `[RETAIN]` | Sole existing mirror writer is the HK approve leg; it must remain fail-visible (F5.3) until its re-termination under the approved rule |
| HKD-02(c) | Mandate's anchor for the verbatim text | (c) lives in the HKD register (Phase 2 authoritative record) |
| HK-006 / HK-008 (EB) | Register-carried; anchors live-verified this session (swallow family; client identity) | The failure classes tests must kill |
| HK-014 (register-carried; `:574–582` verified) & HK-015 | CLEAN-producer chain anchors | F2.3 |
| F13.1 | `[EB, HKD-06/31 + HKD-27 norm; HK-028]` | No fabricated readiness rendering (FO-facing complement) |
| ODI-07 | Hotel operations + engineering | Score calibration pending decision (F5.7 holds) |

**Verified code touchpoints.**

| Touchpoint | Role |
|---|---|
| `apps/api/src/modules/housekeeping/housekeeping.service.ts:560` `approveInspection(id, body)` | Approve writer (to rework) |
| `:574–577` `rooms.update` … `.catch(() => {})` | Swallow cluster 1 (F5.3 target) |
| `:579–582` `room_status` mirror UPDATE `WHERE room_number =$2` (sole mirror writer verification `:580`) | Swallow cluster 2 + `F7.3` sole writer |
| `:586–593` `INSERT INTO room_inspections` + hard-coded score **100** (`:590`) | HKD-27/OQ-11 surface; F5.7 subject |
| `:584` client-supplied `inspectorId`/`parseInt(body.inspectorId)` | HK-008 identity prohibition target |
| `:614` `rejectInspection(id, body)` + note-stitch pattern `|| Rejected:` (`:623–626`) | HK-030/HK-014 surface; F5.2 re-clean production target |
| `housekeeping.controller.ts:285` `@Post('inspections/:id/approve')`, `:290` `@Post('inspections/:id/reject')` | Route layer where WS9 decorators gate (separate workstream) |
| `apps/api/src/modules/front-office/application/commands/check-out/check-out.handler.ts:471` (`rooms.update DIRTY+housekeeping_status DIRTY`) | F2.3/HKD-14 interlock side (FO/Reservations boundary; never modified by HK) |
| `apps/api/src/modules/front-office/check-in/__tests__/room-readiness.spec.ts` | Existing code gate anchor file (`F2.2/F2.8` acceptance home); verified existence only (not executed) |
| Prisma schema (`packages/db/schema.prisma:10959 `room_status`, 10976 config, 10992 history, 3289 `dnd_status`) | F7.1 family truth; `F7.3` no-new-consumer surface |

**Future acceptance-test obligations.**

| Test | Maps from | Trigger/failure | Expected observable |
|---|---|---|---|
| WS3-T1 fail-visible approve | F5.3, HK-006/-008 | Injected DB failure on any of approve's writes (`rooms`/mirror/inspection insert) | HTTP error to caller; NO success body; NO partial-silent state |
| WS3-T2 authenticated identity | F4.5, HK-008 | Submit approve with client-injected `inspectorId` vs none | Client value ignored; authenticated requester recorded |
| WS3-T3 rejection auto re-clean | F5.2 HKD-03 | Reject an inspection | A re-clean task exists targeting the room in the same command; idempotent (no duplicate re-clean targets on repeated rejection) |
| WS3-T4 room condition never written by rejection | F5.2 | Reject with varying reason | `rooms` condition untouched; saleability stays blocked (F5.4) |
| WS3-T5 concurrent decisioning | F5.8 | Parallel approve+reject of one inspection row | First-committed-wins; loser surfaces conflict; no both-write |
| WS3-T6 scoring contract | F5.7/ODI-07 | Approve + reject each | Current contract semantics recorded (100/0 today); calibration change ONLY after ODI-07 decision |
| WS3-T7 READY-never-written | F2.2 | Any approve/reject path after rework | `room-readiness.spec.ts:43–44` acceptance-home passes; no READY write |
| WS3-T8 saleability-invariant | F5.4 | Rejected/pending-resolution room | Not saleable/readiness-blocked everywhere on the FO side; no mixed-state display |

**Entry / exit (future).** *Entry:* EB+APPROVED anchors only — WS3 is gate-free at the fail-visible/re-clean/identity layers, per F7.3's rule. Any mirror-*leg* (wiring-changes) sub-scope remains constrained to finish AFTER this leg is validated independently; HKD-32 removal never in this workstream. *Exit:* WS3-T1…T8 observable; no catch-swallow remains in the inspection write pipeline; auto re-clean exists; authenticated identity everywhere.

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| ODI-07 scoring calibration (current semantics adopted; change is a decision) | Hotel operations + engineering | PENDING DECISION (F5.7 holds; nothing invented here) |
| ODI-11 "reassign vs send-back" consolidation (affects inspection UX layering with WS4) | Engineering lead | PENDING DECISION — both remain normative as written (F4.6/F4.8) |
| HKD-32-related mirror-leg sequencing | Later DB-phase gate (G3) | Sub-scope gate; **not** a Phase 4 blocker; no removal-safety claim anywhere |

**Ratification status:** none (no bundle-locked anchors; HKD-02/03 already in APPROVED/EB authority). `PENDING` only on ODI-07/ODI-11 sub-scopes above.

*Planned tests have not been executed under Phase 4. Phase 4 completion ≠ build authorization.*

---

## 5. WS4 — Staffing, cards & reassignment correction

**Purpose.** Realize the specified staffing model (`[DEL, HKD-23]` — sub-scopes pending bundle execution): attendants+sections as the single staffing truth; day-plan cards for identity/handover only; attendant provisioning writer; FK-backed identity joins; reassignment that preserves `assigned_to` as the attendant's work ownership; move-room wired only with its invariants intact.

**In scope (future build).** Attend provisioning writer (F6.5 — zero-writer gap); staff identity FK joins (F6.6; `housekeeping_attendants.employee_id → employees.id`); card JSON shape per ODI-03 (F6.3 plan-not-work-state); `F6.4` reassign correction over `:806–832`: supervisor ids written on supervisor cards only; task `assigned_to` remains the attendant's; move-room integration ONLY under F4.8's invariants (HKD-02 re-entry rules + OOO/OOS axis reads untouched).

**Not in scope.** Teams (F1.5 [DEL HKD-23 retired]; `/housekeeping/teams` dead affordance); true headcount (F1.4/OQ-09); supervisor ROLE columns (F6.2; designation of inspector is part of OQ-12/WS9); inventing card semantics beyond ODI-03; inventing reassignment semantics beyond F6.4/F4.6 (explicit audited act; handover rule F4.6 preserved).

**Verified anchors.**

| Anchor | Authority label | Bearing |
|---|---|---|
| F6.1/F6.2 | `[DEL, HKD-23]` | Staffing truth model |
| F6.3 | `[DEL, HKD-30]` | Cards=plan-not-work-state; shape via ODI-03 |
| F6.4 | `[DEL/EB, HKD-30 + HK-011]` | `reassignInspection` writes supervisor ids on cards only — HK-011's overwrite is specified out |
| F6.5 | `[DEL, HKD-23 consequence + EB HK-024]` | Provisioning write-path requirement |
| F6.6 | `[EB, HK-012]` | FK-backed join law |
| F4.6 | `[EB, HKD-15]` | Reassignment = explicit audited act; no automatic transfers of unfinished work |
| F4.8 | `[EB, HKD-24]` | Move-room = task-level integration; invariants: HKD-02 re-entry rules + OOO/OOS axis reads (Ch 9) |
| F2.5 | availability-axis read-only law (cross-module) | WS4 must not write `room_status` axis values |
| HK-011 (EB) | `reassignInspection` currently writes `assigned_to` on the TASK with the new supervisor's id (`:816`) | Verified defect; re-spec target |
| HK-012 (EB; register-carried) | getStaff join against the business-code column (`:288–310` attendants→`employees.employee_id` mapped lookups) | FK-join obligation |
| HK-024 (register-carried) | Attendant provisioning zero-writer | F6.5 target |
| ODI-03 | Card-shape decision (Engineering lead, owner-visible per OQ-15 closure) | Decision pending |
| ODI-12 | Room-move readiness acceptance (canonical re-entry rule cross-ref) | Sub-constraint in F4.8 move semantics |
| OQ-15 (carried) | Card contract sensitivity noted by owner | Bounded ODI-03 decision context |

**Verified code touchpoints.**

| Touchpoint | Role |
|---|---|
| `apps/api/src/modules/housekeeping/housekeeping.service.ts:211–259` full `hk_room_assignments` CRUD (`:213/221/239/249/255/259`) | Card stack (F6.3 plan data) |
| `:223` `RA-${Date.now()}` id mint | Same id-family obligation as WS1 |
| `:262–279` `moveRoom()` (`:265/269` reads; `:275/279` updates) | Unrouted method (F4.8 "service method exists unrouted" — verified no move route in `housekeeping.controller.ts` 31–409 capture) — wiring + invariants belong here |
| `:806–832` `reassignInspection` (`:814–817` task-level `assigned_to` overwrite; `:819–829` audit entry) | HK-011 defect surface; F6.4 rewrite site |
| `:806` `newSupervisorId` … `:807/828` supervisor-name handling | Supervisor/attendant distinction preserving F6.2 (cards-only) |
| `:288–310` `getStaff` (`:290` attendants findMany, `:293` mapped `employeeIds`, `:296` employees lookups, `:307–308` section_id/hk_status) | HK-012 FK-join target |
| `:327–330` `getSections` + `:75` `housekeeping_sections findFirst` fallback | Section reads (uniform split-of-truth surface) |
| `apps/api/src/modules/housekeeping/housekeeping.controller.ts:209` `@Get('staff')`; `:214` `@Get('rooms')`; `:219` `@Get('sections')`; `:258–273` `room-assignments` CRUD; `:317–332` `supervisor-assignments` CRUD | Read/wiring surface |

**Future acceptance-test obligations.**

| Test | Maps from | Trigger/failure | Expected observable |
|---|---|---|---|
| WS4-T1 FK-joined staff reads | F6.6/HK-012 | Query staff list | Rows derive from `employee_id → employees.id`; section_id + hk_status present only if backed |
| WS4-T2 provisioning writer exists & is audited | F6.5/HK-024 | Create/update an attendant through the admin/UI path | DB row persists; authenticated identity captured per F4.5 |
| WS4-T3 reassign preserves work attribution | F6.4/HK-011 | Reassign an inspection | Task's `assigned_to` unchanged (attendant-owned); only supervisor-role card carries the new id |
| WS4-T4 card read cannot corrupt work state | F6.3/F4.1 | Render/modify a card's rooms array | No work-state derive; `hk_room_assignments` never feeds status transitions/assignments |
| WS4-T5 card plan shape | ODI-03 decision | Shape test | Matches recorded shape if decided; otherwise recorded default held explicitly; nothing invented either way |
| WS4-T6 move-room invariants | F4.8/HKD-02 & F2.5 | Move a room-task (respects re-entry vocabulary + OOO/OOS axis) | No re-entry violation; no axis write; consumer-choice strictness (ODI-12) respected |
| WS4-T7 handover forward-preserve | F4.6 | Unfinished work at handover | Original assignee carried-forward; reassignment remains explicit audited act only |

**Entry / exit (future).** *Entry:* ⚠️ **PENDING RATIFICATION** on all `[DEL, HKD-23]`- and `[DEL/EB, HKD-30]`-locked rules above → bundle G1 must execute before their implementation authority exists (mandate EC-1 "blocks implementation authority, not spec drafting"). *Exit:* provisioning writer live + audited; FK joins everywhere across all listed touchpoints; supervisor-ids-on-cards-only invariant (T3) provably on; task/card separation invariant (T4) provably on; move-room wired only with invariants (T6); no residual teams surface remains (F1.5/F13.3).

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| Bundle G1 execution (HKD-23, HKD-30) — unlocks the [DEL]-locked rules above | Owner (product) | **PENDING RATIFICATION** (EC-1's exact scope) |
| ODI-03 card-shape if cards persist | Engineering lead (owner-visible) | PENDING DECISION (default recorded: cards = plan data, never work-state) |
| ODI-12 room-move acceptance tolerance | Engineering lead (compatibility assessment) | PENDING DECISION (documented per consumer; strictness locked by canonical re-entry rule HKD-02/05) |
| ODI-11 reassign-vs-send-back consolidation | Engineering lead | PENDING DECISION — both remain normative as written |

**Ratification status:** the [DEL]-anchored sub-scopes (HKD-23, HKD-30) = `PENDING RATIFICATION` until bundle execution. The EB defects' repairs (HK-011, HK-012, HK-024) do **not** wait on it (their authority is EB). *(Per §4.6 both layers must be labeled).* Sub-scope-gating only — the WS is NOT fully blocked; proceed-by-label per its recorded state.

*Planned tests have not been executed under Phase 4. Phase 4 completion ≠ build authorization.*

---

## 6. WS5 — Guest requests (room-service authoritative)

**Purpose.** Kill the `Guest:` string protocol and move HK-facing request consumption onto the verified room-service authority (F10.1), with end-to-end persistence + audit (HKD-20 contract). No note-based linkage, no fire-and-forget, no fake request flows before persistence lands.

**In scope (future build).** Retire the note-encoding/regex-parsing/note-sniffing protocol (F4.7); replace with real integration consumption (HK requests via FO-adjacent live routes as already verified in `room-service.controller.ts` route list); persistence + audited linkage for room-service→HK requests (F10.2 "end-to-end audit"); `never silently complete` the source request from the HK side (F10.2 first rule; `fire-and-forget` prohibited); retire web-side `Guest: ${req.type} - ${req.note} [${req.id}]` synthesis; retire HK UI `GuestRequestsSection`'s present synthetic flow once no fake workflows remain (F10.2/HK-001 family; HKD-27 rule).

**Not in scope.** A competing HK-owned request lifecycle (F10.1 single authority); mobile surface (F13.4/HK-025); inventing the room-service persistence schema (the contract's underlying model is a design-time verification item — the routes exist today but persistence semantics are NOT this workstream's invention); `sla` overdue engine (F10.3/HKD-12 display-only).

**Verified anchors.**

| Anchor | Authority label | Bearing |
|---|---|---|
| F10.1 | `[DEL, HKD-20 register text]` "room-service authoritative; HK string protocol retired in spec; HK requests via FO-adjacent live routes; persistence+audit required end-to-end" | Head rule |
| F10.2 | `[DEL/EB, HKD-20 + HK-030]` | No silent-completion; "must not present workable fake request flows" until real persistence lands |
| F4.7 | `[DEL, HKD-20 + HK-030]` | Protocol family retired spec-wide (string encoding, regex-parse, note-sniffing, fire-and-forget, direct task PATCH from GuestRequests) |
| F15.2 | `[EB, per-boundary rows]` | Producer-outage surfaces as consumer error, never silent degradation |
| F4.5 | authenticated identity on all request-linkage mutations | Standard |
| F13.3 | `[EB/DEL mixture per record; HK-001]` | request endpoints dead-URL family from HK-001's Annex-A → retire/integrate disposition |
| HK-030 (EB) | Note protocol active from BOTH web UI (`GuestRequestsSection.tsx:155/:387`) and server reject-side note-stitch (`:623–626`) | Verified defect; replacement obligation |
| HK-001 (register-carried; request-family parts) | Dead HK request endpoint URL family | F13.3 disposition holds |
| HKD-20 | Bundle member | Label PENDING RATIFICATION sub-scope |

**Verified code touchpoints.**

| Touchpoint | Role |
|---|---|
| `apps/web/app/(dashboard)/housekeeping/_components/GuestRequestsSection.tsx:155/:387` | The `Guest:` synthesis web-sides — verified literal protocol surface |
| `apps/api/src/modules/room-service/room-service.controller.ts:10–59` — `@Get('requests') :10`, `@Get('departments') :19`, `@Get('room-guest/:roomNumber') :24`, `@Get('request-types') :29`, `@Post('requests') :34`, `@Put('acknowledge') :39`, `@Put('assign') :44`, `@Put('complete') :49`, `@Put('escalate') :54`, `@Delete('requests/:id') :59` | **The authority — verified live routes today** (consumption + creation endpoints exist; map here is the integration map) |
| `apps/api/src/modules/room-service/room-service.service.ts`, `room-service.module.ts`, `alert.store.ts` (4 files total; module existence verified) | Source-side of the contract; persistence semantics to be confirmed per-contract at integration design (UNVERIFIED lower layer) |
| `apps/api/src/modules/housekeeping/housekeeping.service.ts:165–174` — the `Guest:` exclusion subset inside the destructive reset (the `contains 'Guest:'` filter) | The note-sniffing surface verified live this session; the protocol's other surfaces (web mint, reject-side stitch) are register-carried from HK-030 and verified at their own touchpoints below/above (F4.7 removes the whole family) |
| `apps/api/src/modules/housekeeping/housekeeping.controller.ts` — no `/requests` routes in verified capture | Confirms the dead-URL family (F13.3 disposition) |

**Future acceptance-test obligations.**

| Test | Maps from | Trigger/failure | Expected observable |
|---|---|---|---|
| WS5-T1 note protocol gone | F4.7/F10.1 | Any legacy `Guest:` shaped note passage through HK tasks / requests / generation / UI | Zero occurrences (session-wide); the `Guest:` mint never runs again |
| WS5-T2 authoritative linkage | F10.1 | HK-ward consumption through `room-service.controller` requests route | Data lands in `room-service` request tables (or its authoritative model) with no HK-owned dual lifecycle |
| WS5-T3 end-to-end audit | F10.2/F4.5 | HK-side linkage mutation | Authenticated identity + reason + timestamp retained; complete lineage |
| WS5-T4 no silent completion | F10.2 HKD-20 | HK task completes without room-service completion | Source request unchanged from HK-side path; no fake closure |
| WS5-T5 no fake request flows | HKD-27/F10.2 | Pre-persistence state | The HK UI does not expose request workflows as workable before the contract's persistence exists (dead affordances removed per F13.3) |
| WS5-T6 persistence semantics honesty | F15.2 (producer-outage) | room-service outage | HK surfaces the error; no silent degradation |

**Entry / exit (future).** *Entry:* ⚠️ the HKD-20-locked rules are within the ratification bundle — their implementation authority activates after the bundle executes (per EC-1). The room-service contract's **persistence semantics must be design-verified** at integration build (the routes/names verified here; the data layer is UNVERIFIED beneath, and WS5 must not resort to a naming guess: record as a design-time verification step with owner = engineering lead). *Exit:* no `Guest:` string remains across HK-side UI + API + generation; request linkage via room-service authority requires complete audit lineage before release; web fake-flow affordances retired/dead-endpoints removed.

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| Bundle G1 execution (HKD-20) | Owner (product) | **PENDING RATIFICATION** |
| Room-service contract's persistence/audit contract details (data-layer verification) | Engineering lead | PENDING VERIFICATION (module + routes verified; model/UI contract UNVERIFIED beneath routes) |
| OQ-14/F1.6 (supplies-facing UI scheduling) — adjacent but NOT a WS5 gate | Owner (case-by-case at the UI phase) | No block |

**Ratification status:** the `[DEL, HKD-20]`-anchored sub-scopes = `PENDING RATIFICATION` until bundle execution; the HK-001 dead-URL dispositions (HKD-27/28 family) proceed on their own authority. *(Per §4.6 both layers must be labeled).*

*Planned tests have not been executed under Phase 4. Phase 4 completion ≠ build authorization.*

---

## 7. WS6 — Linen (real persistence; syntheses prohibited)

**Purpose.** Turn linen from a fabricated-data display into a required capability with real persistence (F11.1), eliminating the two documented syntheses (HK-002 client-side refill loop; HK-035 fixed-ratio location splits).

**In scope (future build).** Issue/return/damage flows specified at contract level with end-to-end audit (F11.1); persistence of shown numbers using the recorded `linen_inventory` resource (server already reads real stock fields); removal of the synthesized movements and fixed-ratio splits (F11.2); linen screens reading real data (F13.3); `getLinenDashboard` counter truthfulness under F12.2's norm.

**Not in scope.** Laundry (F11.3 `[PROV, HKD-22]` excluded; separate capability study only on proof of in-house operation); operational parameters (rounds cadence/issue authorization/damage disposition stream — ODI-09, hotel operations, nothing invented); par-location model choice beyond ops input (ODI-06); HK-facing supplies view (F11.5/F1.6, OQ-14 case-by-case); inventory/procurement lifecycle ownership (F11.4: inventory/procurement domain owns it; HK orders through it — HKD-29 recorded as an inventory-spec input, not an HK workstream).

**Verified anchors.**

| Anchor | Authority label | Bearing |
|---|---|---|
| F11.1 | `[DEL, HKD-21]` | Required capability with real persistence; `linen_inventory` resource recorded; flows contract-level |
| F11.2 | `[DEL/EB, HKD-21 + HKD-27 norm]` | Synthesized refill loop (HK-002) and fixed-ratio splits (HK-035) prohibited; missing data displays as missing |
| F11.3 | `[PROV, HKD-22]` | Laundry excluded (PENDING RATIFICATION label preserved; exclusion stands regardless) |
| F11.4 | `[APPROVED verbatim, HKD-28 + DEF, HKD-29]` | Supply lifecycle separation; HK stack incl. dead approve (HK-037) retired |
| F11.5 | `[DEF, OQ-14]` | No HK supplies-view obligation |
| F13.3 | `[EB/DEL mixture; HK-001]` | Dead linen-write URL family → retire/disposition per F1.9 |
| HK-002 / HK-035 (EB) | Register-carried syntheses; UI fixed-vocab verified at `LinenSection.tsx:20/46/57–58`; client refill-loop callsites not re-grepped this session | F11.2 target classes |
| HK-037 (EB) | Supply approve transition dead (approve endpoint can only 400) — transitions map verified `:8–9/44–47`; dead-end conclusion register-carried | F11.4's documented stack state |
| ODI-06 / ODI-09 | Hotel operations (+ engineering for ODI-06) | Operational inputs pending — nothing invented |

**Verified code touchpoints.**

| Touchpoint | Role |
|---|---|
| `apps/api/src/modules/housekeeping/housekeeping-inventory.service.ts:72–80/128/235–236` (stock value / out-of-stock / low-stock / safety / overstock / totals over `quantity_on_hand` state) | **Real-data server reads verified** (F11.1's "server already reads" claim holds) |
| `apps/api/src/modules/housekeeping/housekeeping-stock.service.ts:360` (expiry-class derivation) | Stock domain reads |
| `apps/api/src/modules/housekeeping/housekeeping.controller.ts:71/76/81/86` `@Get('linen') /types /locations /linen/dashboard` | Read surface (GET-only; no linen write routes in the verified 62-route capture → HK-001 dead-write family confirmed structurally) |
| `:387` `getLinenDashboard()` | Dashboard read (WS8-complementary) |
| `apps/web/app/(dashboard)/housekeeping/_components/LinenSection.tsx:20/46/57–58` | UI-side fixed vocabulary maps (synthesis-adjacent); HK-002's client refill loop itself is register-carried and not individually re-grepped this session |
| Recorded resource label `linen_inventory` (HKD-21/F11.1) | Table-name mapping in current schema NOT re-established this session (ERR-E) → WS6 design-verification item |

**Future acceptance-test obligations.**

| Test | Maps from | Trigger/failure | Expected observable |
|---|---|---|---|
| WS6-T1 real-data-only reads | F11.1/F11.2 | Render linen screens per location | Every displayed number traces to persisted rows; missing data displays as missing (never a fabrication) |
| WS6-T2 no synthesized movements | F11.2/HK-002 | Load/operate any linen view (incl. client refill path) | Zero client-side stock mutations; all writes server-side persistence |
| WS6-T3 no fixed-ratio splits | F11.2/HK-035 | Location totals | Splits derive from persisted location data (ODI-06 model), never a client-fixed ratio |
| WS6-T4 audited issue/return/damage | F11.1 | Each flow mutation | Persisted, with authenticated identity + timestamp + full lineage (F4.5) |
| WS6-T5 dead-write endpoints absent | F13.3/HK-001 | Probe `:71–86` family beyond GETs | No fabricated write affordance anywhere (UI or API) |
| WS6-T6 supply-stack separation | F11.4/HKD-29 | Cross-module touch on supply state | HK never mutates the inventory/procurement-supplied lifecycle state; HK orders only through it |

**Entry / exit (future).** *Entry:* ⚠️ the `[DEL, HKD-21]` rules sit in the ratification bundle — implementation authority activates after G1 executes (EC-1); table-name mapping (ERR-E) design-verified first; ODI-06/09 answered by hotel operations. *Exit:* WS6-T1…T6 observable; no synthesis remains; persistence + audit end-to-end.

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| Bundle G1 execution (HKD-21) | Owner (product) | **PENDING RATIFICATION** |
| ODI-06 par/location alignment | Hotel operations (with engineering) | PENDING DECISION |
| ODI-09 operational workflow parameters (rounds, cadence, authorization, damage stream) | Hotel operations | PENDING DECISION |
| `linen_inventory` table-name mapping in current schema | Engineering lead (design-time) | UNVERIFIED detail (ERR-E) |
| HKD-29 (inventory-spec input) | Inventory domain owner | Boundary note — not an HK gate |

**Ratification status:** `[DEL, HKD-21]`-locked sub-scopes = `PENDING RATIFICATION`; `F11.3 [PROV, HKD-22]` label preserved verbatim (out of scope regardless).

*Planned tests have not been executed under Phase 4. Phase 4 completion ≠ build authorization.*

---

## 8. WS7 — Blocks consumption (OOO/OOS respect; axis read-only)

**Purpose.** Make HK generation and assignment respect the maintenance-owned OOO/OOS block state during generation and assignment, and prevent any HK write to the axis — per F9.1/F9.2's authority split and F2.5's read-only law.

**In scope (future build).** Block-state consumption at generation (blocks respected — blocked rooms not worked); respect at assignment; the HK **request** path (HK may request a block, never place/release — F9.1); literal normalization at the consumption boundary only (F9.4: maintenance contract defines the vocabulary; the FO `RoomStatus` `'OOO'` note is an FO boundary, not an HK rule); readiness composition stays `condition × availability × block` documented, never merged (F2.5/F2.8 family).

**Not in scope.** Building or assuming the maintenance lifecycle itself (owner capability is UNVERIFIED in-code this session — ERR-B); inventing the maintenance contract or schedule dependency (blocked-on); placement-authorization role design (ODI-05, hotel operations); writing the availability axis (F2.5/F9.2 — hard prohibition).

**Verified anchors.**

| Anchor | Authority label | Bearing |
|---|---|---|
| F9.1 | `[DEL, HKD-04]` | Maintenance owns lifecycle end-to-end; HK consumer/requester only |
| F9.2 | `[DEL+EB, HKD-04 + HKD-31]` | Generation and assignment respect block state; HK never writes the axis |
| F9.3 | `[OPEN/DEL, ODI-05]` | Placement-auth roles = operations process question; contract names owner; role matrix is ops input |
| F9.4 | `[EB, HK-019/HK-20 boundary]` | Literal fragmentation hazard; per-contract normalization only |
| F2.5 | axis ownership law | Availability axis read-only for HK |
| F3.5 (HKD-14) | checkout-adjacent task creation | Generation linkage where block-respect applies |
| HK-019 (EB) | Literal fragmentation across `{OUT_OF_ORDER|OOO|OOOS|OOS|OUT_OF_SERVICE}` readers | F9.4 target |
| HK-020 (EB) | HK tasks blind to maintenance blocks— verified structurally: `generateDailyTasks` has **no block filter** (`:162–207` inspected) | F9.2 target |
| ODI-05 | Persisted in `phase-3 03:16` | Hotel-ops decision pending |

**Verified code touchpoints.**

| Touchpoint | Role |
|---|---|
| `apps/api/src/modules/housekeeping/housekeeping.service.ts:162–207` `generateDailyTasks()` | Generation surface where block-respect must compose (today: no block filter — HK-020 structural verification) |
| `apps/api/src/modules/housekeeping/housekeeping.controller.ts` + `hk_room_assignments` surfaces (`:211–259`) | Assignment consumption surface |
| `apps/api/src/modules/engineering-maintenance/` (3 files: controller/module/service) | **Owner-side: exists, but bounded grep `OUT_OF_ORDER|out_of_order|OOO` → zero matches — lifecycle capability not located in-code (ERR-B); no contract may be invented** |
| `apps/api/src/modules/front-office/application/commands/check-out/check-out.handler.ts:471` | Axis-adjacent `rooms` column writes by Reservations/FO (F-25-family posture: Reservations writes `rooms` columns — HK mirror writes are the only other axis touch, reading-only except the F5.3-controlled approve leg) |

**Future acceptance-test obligations.**

| Test | Maps from | Trigger/failure | Expected observable |
|---|---|---|---|
| WS7-T1 block-respecting generation | F9.2/HK-020 | Generate with an OOO/OOS-blocked room present | No task for the blocked room; no work assignment against it |
| WS7-T2 no HK axis writes | F2.5/F9.2 | Any HK mutation | Zero availability-axis writes from HK anywhere (compact test over all HK mutation paths) |
| WS7-T3 request path authority | F9.1 | HK-initiated block request | Flow goes to maintenance ownership; HK cannot place/release |
| WS7-T4 vocabulary normalization | F9.4/HK-019 | Literal-shaped inputs across readers | Normalized exactly per the maintenance contract (no HK-local literal innovation) |
| WS7-T5 readiness composition integrity | F2.5/F2.8 | Gate evaluation on a blocked room | Condition × availability × block composition documented; never merged into one axis |

**Entry / exit (future).** *Entry:* ⚠️ `[DEL, HKD-04]` rules → PENDING RATIFICATION sub-scopes; the maintenance-domain lifecycle in code must exist and expose its contract (owner capability — currently UNVERIFIED/UNBUILT per ERR-B; any dependency is declared as future cross-domain work, not assumed); ODI-05 answered by hotel operations. *Exit:* WS7-T1…T5 observable; zero axis writes; blind-spot (HK-020) dead.

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| Maintenance-domain lifecycle owner capability in code + lifecycle/schedule contract | Maintenance domain owner (engineering) | **UNVERIFIED / UNBUILT (ERR-B)** — declared dependency; no maintenance contract invented |
| Bundle G1 execution (HKD-04) | Owner (product) | **PENDING RATIFICATION** |
| ODI-05 placement-authorization roles | Hotel operations (owner-delegated queue) | PENDING DECISION |

**Ratification status:** `[DEL, HKD-04]`-locked sub-scopes = `PENDING RATIFICATION`.

*Planned tests have not been executed under Phase 4. Phase 4 completion ≠ build authorization.*

---

## 9. WS8 — Dashboards & UI truthfulness

**Purpose.** Enforce server-authoritative counters and the truthfulness family (F12/F13), secure the ODI-08 endpoint decision, and prepare verified retirement candidates (F12.4/F1.9) for their own later-phase gates.

**In scope (future build).** Server-authoritative counters (F12.1 — implemented dashboard endpoint is the ready candidate **pending ODI-08**); fabricated counters/banner/pseudo-task/SLA/spinner/dead-button removals (F12.2/HK-022); audits-history refresh contract (F12.3/HK-033); missing-data renders as missing (F13.1); axes separation (F13.2/HK-029); no endpoints-called-that-don't-exist; verified dead-`_components` retirement candidates list (F12.4/F1.9: crash-if-render `MinibarSection`, duplicate lost-found affordances, dead move-room UI call); **no new mirror consumers** (F7.3 — dashboards must read `rooms`/domain, never the mirror).

**Not in scope.** Deciding ODI-08 by planning (owner decision); executing any retirement in this phase (F1.10: executions in later phases' own gates); mobile surface (F13.4/HK-025); no mirror legitimacy.

**Verified anchors.**

| Anchor | Authority label | Bearing |
|---|---|---|
| F12.1 | `[EB, HKD-27]` | Server-computed counters verified as the ready candidate; HKD-27's OQ closure recorded "unused endpoint is the ready candidate (or it is removed)" |
| F12.2 | `[EB, HKD-27 + OQ-13 closure]` | Fabrications specified out |
| F12.3 | `[EB, HK-033]` | History-refresh consumption rule |
| F12.4 | `[EB, HKD-24/25/26]` | move-room/lost-found/six-dead-`_components` dispositions |
| F13.1/F13.2/F13.3 | `[EB, HKD-06/31 + HKD-27; HK-028]`, `[EB, HKD-06/31/01; HK-029]`, `[EB/DEL mixture; HK-001]` | Truthfulness trio |
| F1.9/F1.10 | `[APPROVED verbatim, HKD-19]` / `[RETAIN/EB/DEL mixture per record]` | Case-by-case + execution-in-own-gates |
| HK-022 (EB; register-carried enumerations) | Fabricated counters, UV ratings, pseudo-task synthesis, "AI Route Optimization Active" banner (verified live at `DashboardSection.tsx:336`), fake spinners, dead Reassign button | F12.2 target classes |
| HK-023 (EB) | Six dead `_components` incl. crash-if-render `MinibarSection` | Verified: file exists (`_components/MinibarSection.tsx`, default-export) |
| HK-028/HK-029 (EB) | Fabricated CLEAN display; grid axis-mixing | F13.1/F13.2 targets; specific FO rack callsites register-carried (FO-owned defect sites — FO-domain frame) |
| HK-033 (EB) | History stale-after-approve until re-entry | F12.3; UI touchpoint group not re-grepped individually this session → marked UNVERIFIED at file level |
| HK-034 (EB; route verified) | `GET /housekeeping/dashboard` unused (register-carried claim) | F12.1/ODI-08 subject |
| ODI-08 | Owner at review cluster (HKD-27 chapter); engineering detail | Adopt-or-remove decision pending |
| HKD-24/25/26 | Their own recorded labels (UI-phase obligations, per F12.4) | Capability-disposition anchors |

**Verified code touchpoints.**

| Touchpoint | Role |
|---|---|
| `apps/api/src/modules/housekeeping/housekeeping.service.ts:353–355` (`getDashboard()`) and `:387 getLinenDashboard()` | Server counter surfaces |
| `housekeeping.controller.ts:66 @Get('dashboard')` | ODI-08 endpoint under decision |
| `apps/web/app/(dashboard)/housekeeping/page.tsx:9/20/62` (imports + tab incl. `activeTab === 'linen'` → LinenSection) | Composition surface incl. the tab map listing `Linen & Laundry` |
| `apps/web/app/(dashboard)/housekeeping/_components/DashboardSection.tsx:336` `"AI Route Optimization Active"` | F12.2's named fabrication — verified live |
| `apps/web/app/(dashboard)/housekeeping/_components/MinibarSection.tsx` | F12.4's crash-if-render retirement candidate — verified present |
| `apps/web/app/(dashboard)/housekeeping/_components/GuestRequestsSection.tsx:155/387` | (cross-ref WS5) — request synthesis family |
| HK-022's additional call-sites / HK-033's history leg / FO rack sites | UNVERIFIED file-level this session (register-carried) — WS8 build must enumerate+verify each before touching |

**Future acceptance-test obligations.**

| Test | Maps from | Trigger/failure | Expected observable |
|---|---|---|---|
| WS8-T1 server-authoritative counters | F12.1 | ODI-08-adopted endpoint + UI render | Every counter traces to a server-computed source; no client-synthesized values |
| WS8-T2 fabricated-artifact absence | F12.2/HK-022 | Load dashboard family | Zero fabricated banners/spinners/pseudo-tasks/SLA values/dead buttons |
| WS8-T3 history refresh contract | F12.3/HK-033 | Approve → open history without re-entry | Contract-specified behavior honored; no stale invention |
| WS8-T4 missing-data renders as missing | F13.1/HK-028 | Absent `housekeeping_status` | "not ready/unknown" — never `CLEAN` |
| WS8-T5 axes separation | F13.2/HK-029 | Render grids per axis | Condition/occupancy/block from their own owners, never a mixed derive |
| WS8-T6 dead affordances gone | F13.3/F12.4/HK-001 | Drive UI incl. `MinibarSection` family crashes candidate | No dead URL calls; retirement candidates removed only under their own later-phase gate |
| WS8-T7 no new mirror consumers | F7.3 | Any dashboard query | `rooms`-column reads only; zero mirror-family reads introduced |

**Entry / exit (future).** *Entry:* ODI-08 decided by the owner (adopt-or-remove — endpoint choice, not calendar); counter truth is gate-free on EB anchors; retirement executions strictly inside their later-phase gates (F1.10). *Exit:* WS8-T1…T7 observable; ODI-08 recorded as executed per owner evidence; no mirror legitimacy anywhere.

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| ODI-08 adopt-or-remove decision | Owner at review cluster (HKD-27 chapter) | PENDING DECISION |
| Retirement-execution gates for each listed candidate | Per HKD-28/F1.10 discipline (later phases' own gates) | Not Phase 4 — preserved принцип |
| HK-022-family call-site enumeration + verification | Engineering lead (build-time) | UNVERIFIED file-level (register-carried enumerations) |
| HK-033 history-leg file-level verification | Engineering lead (build-time) | UNVERIFIED file-level |

**Ratification status:** no bundle-locked anchors (HKD-24/25/26/27 are not bundle members; HKD-19-family dispositions are APPROVED-verbatim).

*Planned tests have not been executed under Phase 4. Phase 4 completion ≠ build authorization.*

---

## 10. WS9 — Authorization (HARD-BLOCKED on OQ-12)

**Purpose.** Activate the role→action matrix and route decorators for the 62 HK routes **once and only once** the OQ-12 sign-off exists; plan the four remaining F14.5 determinations for that signature act; do not propagate the block to any other workstream.

**In scope (future build, post-sign-off).** Route decorators for the 62 routes per F14.5(b); `housekeeping:*` registry definitions + grants per F14.5(a); the guard-system choice per F14.5(c) (common `@Permissions` vs platform `@Permission` — ADMIN-bypass semantics differ); the inspector-role determination per F14.5(d) (HOUSEKEEPING_MANAGER-only vs a new HOUSEKEEPING_SUPERVISOR); retired-stack routes get NO verb adoption (F14.6 — the HK discrepancy/mobile family's routes remain pending stack retirement, `07 §9.3` gap 5).

**Not in scope.** Any activation without the written OQ-12 sign-off (hard gate, non-delegable); inventing the matrix (F14.4 is the recorded intended policy, ratification-ready as-is); propagating the WS9 block elsewhere (mandate §4.2); choices extending beyond the recorded matrix.

**Verified anchors.**

| Anchor | Authority label | Bearing |
|---|---|---|
| F14.1/F14.2 | `[verified current state; 07 §9.1]` | Chain (JWT + property-scope + multi-tenant globals + two metadata-driven permission guards + an unused RolesGuard): every guard passes any route lacking its decorator; **all 62 HK routes carry no role/permission decorators** — implemented fact |
| F14.3 | `[verified current state; 07 §9.1]` | HK department roles currently receive only `command-center:*`/`common:*` grants; `housekeeping:*` codes consumed-but-never-defined-or-granted; no `@Roles(` decorator anywhere in the API |
| F14.4 | `[INTENDED — OPEN RATIFICATION, matrix from 07 §9.2]` | SUPER_ADMIN/ADMIN bypass; HOUSEKEEPING_MANAGER: generate/approve-reject/reassign/config read-write/dashboard-audit reads/linen ops; HOUSEKEEPING_STAFF: own task start/complete + own plan; FRONT_DESK: none (FO covers consumption); other departments: none |
| F14.5 | `[OPEN determination]` | The four activation items (a)–(d) listed above |
| F14.6 | `[DEL, HKD-16-family]` | Retired stacks need no verb adoption |
| HK-007 (EB + register-carried matrix) | The authorization-openness finding; full matrix investigation record `07 §9` incl. `:45/:51/:73/:24` code citations | Defect posture |
| OQ-12 / ODI-10 | Security owner + product owner — **explicit human sign-off, non-delegable** | G2 (mandate §4.2: blocks WS9 only) |

**Verified code touchpoints.**

| Touchpoint | Role |
|---|---|
| `apps/api/src/common/authorization/permission.guard.ts` + `permissions.decorator.ts` | Common `@Permissions` system (metadata-driven pass-through at `:45`; ADMIN/SUPER_ADMIN bypass at `:51`; fail-closed deny at `:73` per `07 §9.1`) |
| `apps/api/src/common/authorization/roles.guard.ts` | RolesGuard (unused per F14.3 — presence verified this session; ERR-A target) |
| `apps/api/src/platform/permission/permission.guard.ts` + `permission.decorator.ts` + `permission-resolver.service.ts` + `permission.service.ts` + `permission-registry.service.ts` + `permission-cache.service.ts` + `permission.controller.ts` + `permission.module.ts` | Platform `@Permission` system (verified file set; `:24` pass-through per `07 §9.1`) |
| `apps/api/src/platform/registry/permission-registry-init.service.ts:358/:372` (`HOUSEKEEPING_MANAGER` / `HOUSEKEEPING_STAFF`) | Role registry (verified sites) |
| `apps/api/src/common/authorization/__tests__/permission-guard-hydration.spec.ts:180`; `apps/api/src/platform/registry/__tests__/permission-registry-transfer-seed.spec.ts:109` | Existing test artifacts around housekeeping roles — inspected existence only; no test executed |
| `apps/api/src/modules/housekeeping/housekeeping.controller.ts` — 62 method decorators verified | The decorated surface (F14.2's "no decorators" verified: zero role/permission decorators in the same capture) |

**Future acceptance-test obligations.**

| Test | Maps from | Trigger/failure | Expected observable |
|---|---|---|---|
| WS9-T1 matrix binding | F14.4/F14.2, HK-007 | Per-role call to each HK route | Exact matrix behavior: MANAGER allowed / STAFF scoped-to-own / FRONT_DESK 403 / other departments 403 |
| WS9-T2 decorator completeness | F14.5(b) | Audit the 62 routes | Every route carries exactly the decided decorator; none missed |
| WS9-T3 grants & registry defs | F14.5(a)/F14.3 | Registry/catalog check | `housekeeping:*` codes defined AND granted; no orphan grants |
| WS9-T4 guard-system single truth | F14.5(c) | Both guard systems exercised | Decided system active; current ADMIN-bypass semantics difference reconciled per sign-off record |
| WS9-T5 inspector-role binding | F14.5(d) | Inspection approve/reject per signed-off role | Decision's role enforced; not via invented alternative |
| WS9-T6 bypass checks | F14.1/F14.2 | Non-decorated-role users incl. legacy paths | No accidental privilege; fail-closed per deployed guard semantics |
| WS9-T7 retired stacks excluded | F14.6 | Registry review | No verbs adopted for the discrepancy/mobile family |

**Entry / exit (future).** *Entry:* **hard gate — OQ-12 sign-off evidence (security owner + product owner, non-delegable)** must be recorded; the four F14.5 determinations answered in the sign-off; guard-system choice verified (not guessed). *Exit:* WS9-T1…T7 observable pre-release; the matrix's exact sign-off record carried verbatim into implementation.

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| **OQ-12 sign-off** (G2) | Security owner + product owner | **BLOCKED (hard, non-delegable)** — OQ-12 = OPEN |
| F14.5(a)(b)(c)(d) determinations | Security owner + product owner (inside the sign-off) | OPEN determinations |
| Guard-system decision | Inside the sign-off | OPEN |

**Ratification status:** exactly this workstream IS the ratification subject; it remains `BLOCKED` until the sign-off exists — and this block must not propagate to any other workstream (mandate §4.2).

*Planned tests have not been executed under Phase 4. Phase 4 completion ≠ build authorization. WS9 activation with No OQ-12 sign-off would itself violate G2 — it cannot be planned as running in parallel.*

---

## 11. WS10 — Stack retirement executions (deferred; own-gate discipline)

**Purpose.** Hold the retirement ledger's later-phase executions of the retired/ineligible stacks verified as CQRS-domain surface, HK supply stack, string protocol, dead affordances — nothing executes in Phase 4.

**In scope (FUTURE-PHASE executions, each under its own gate — not Phase 4).** Person-discrepancy FO-authority retirement of the HK-tagged stack (F8.1/F8.4: the missing-table `person_discrepancies` surface, the `GET /housekeeping/discrepancies` 404-family, `/mobile/discrepancies` read/write family, and person-count writes — no verb adoption, no product behavior); supply-stack retirement (F11.4/HKD-28: HK-tagged stack dispositionally retired, incl. the dead approval transition HK-037); note-protocol retirement (F4.7/F10.1/HKD-20); dead affordances by per-record disposition (F1.9/F1.10: `MinibarSection` crash-if-render family, duplicate lost-found affordances, dead move-room UI call); zero-consumer legacy table family (F7.5: `room_status_config`/`dnd_status` excluded from the specification entirely). No mobile surface rises despite any family mention (F13.4/HK-025 preserved).

**Not in scope.** Any retirement execution in Phase 4; silent resurrection of anything (F1.10); qualifying retirement elsewhere's authority (each stays in this workstream's ledger with its own trigger); resurrecting a mobile stack (no mobile surface, F13.4 — this is the disposition rule against related-stack keepers).

**Verified anchors.**

| Anchor | Authority label | Bearing |
|---|---|---|
| F8.1 | `[APPROVED verbatim, HKD-16]` | Person discrepancy: FO single authority; HK-tagged stack dispositionally retired |
| F8.4 | `[DEL/EB, HKD-16 family; HK-003]` | The exact retired family incl. `/mobile/discrepancies`; "nothing in this spec defines them" |
| F8.5 | `[queue closure OQ-09]` | Headcount stays outside (guardrail) |
| F11.4 | `[APPROVED verbatim, HKD-28 + DEF, HKD-29]` | Supply-stack retirement; retirement in later phases' own gates (HKD-28 binding) |
| F4.7/F10.1 | `[DEL, HKD-20 + HK-030]` | String-protocol retirement (WS5 owns the replacement path) |
| F13.3/F7.5/F1.9/F1.10 | `[EB/DEL mixture; HK-001]` / `[EB, HKD-07/08]` / `[APPROVED verbatim, HKD-19]` / `[RETAIN/EB/DEL mixture per record]` | Dead URLs; zero-consumer tables (HKD-07/08); case-by-case; own-gates |
| HK-003 (register-carried) | HK stack never-created-table `person_discrepancies` | F8.4's anchor |
| HK-030 / HK-037 | Note-protocol family / supply-dead-end | WS5/F11.4 anchors, same retirement ledger |
| HKD-28 | Retirement-governance rule | Binding execution discipline |
| OQ-14 (F1.6) | Case-by-case UI item only | Guards supplies-view |

**Verified code touchpoints.**

| Touchpoint | Role |
|---|---|
| `apps/api/src/modules/housekeeping/application/commands/bulk-resolve-discrepancies.command.ts` + `.handler.ts`; `record-hk-person-count.command.ts` + `.handler.ts`; `domain/entities/person-discrepancy.ts`; `domain/repositories/person-discrepancy.repository.interface.ts`; `infrastructure/repositories/person-discrepancy.repository.ts`; `dto/stock.dto.ts`/`dto/supply-request.dto.ts`; `events/housekeeping.events.ts` | The untracked restructure-layer supply (F8.4 family; all pre-existing user work — verified via tree listing) |
| `housekeeping.controller.ts:339` `@Put('rooms/:roomNumber/person-count')`; `:357 @Get('mobile/discrepancies')`; `:373` `@Put('mobile/rooms/:roomNumber/person-count')`; `:389` `@Post('discrepancies/:id/resolve')`; `:409` `@Post('discrepancies/bulk-resolve')` | Retired-family routes (no verb adoption; F8.4) |
| Supply routes `:117–202` full verified capture; `housekeeping-supply-request.service.ts:8–9/44–47/64` transition map + guard + history insert; `:846 `SUP-${Date.now()}`` | HK supply stack family |
| Note-protocol surfaces (web `GuestRequestsSection.tsx`, reject stitch `:623–626`, `Guest:` filter `:165–171`) | Protocol family (WS5's cross-ref) |
| `MinibarSection.tsx` (verified present) | Crash-if-render retirement candidate (F12.4/HKD-26) |

**Future acceptance-test obligations.** These are per-retirement, each executed **within its own later-phase gate** at its own phase (per HKD-28 and F1.10 — not established here): the retirement ledger table in `02` §6 carries its evidence column. No Phase 4 acceptance-test is claimed for any retirement execution.

**Entry / exit (future).** *Entry:* each retirement runs under its OWN later-phase gate and discipline, sequenced after the phases' gates; **no prepare-and-wait obligation exists in Phase 4.** *Exit:* none applicable; the ledger persists for later-phase reference with zero claims of execution.

**blocked-on.**

| Input | Owner | Status |
|---|---|---|
| Each retirement's own gate (HKD-28 discipline; F1.10; G4 where reopeners apply) | Later-phase owners per its own gate | Not applicable in Phase 4 (no execution, no gate) |
| Any missing ownership boundary for a target | Later-phase owner identification | Record as discrepancy *if it arises at the retirement phase* (none inventoried here) |

**Ratification status:** full retirement-ledger authority sits with F1.10/HKD-28's recorded governance; no bundle execution is required for *this* workstream's non-execution status (its "W4-after-own-gates" placement is planning-structure, not authority).

*Planned tests have not been executed under Phase 4. No retirement executes in Phase 4. Phase 4 completion ≠ build authorization.*

---

## 12. Sequencing (relative order only — no dates, no effort estimates, no completion commitments)

### Wave 0 — Decisions (unlock key inputs; no build authorization implied)

| Decision set | Owner (recorded) | Unlocks |
|---|---|---|---|
| Ratification bundle G1 (HKD-04/06/20/21/23/30 + HKD-10/22) | Owner (product) | WS4 `[DEL,HKD-23/30]`-locked sub-scopes; WS5; WS6; WS7 scopes |
| OQ-12 sign-off, incl. the four F14.5 determinations (ODI-10) | Security owner + product owner (non-delegable) | WS9 activation (and ONLY WS9) |
| ODI-01, ODI-02 (business-date key; naming/merge) | Engineering lead | WS1 |
| ODI-03 (card shape), ODI-11 (reassign-vs-send-back), ODI-12 (move acceptance tolerance) | Engineering lead | WS4 (+ WS3 border) |
| ODI-06, ODI-09 (par alignment; linen operational params) | Hotel operations | WS6 |
| ODI-05 (block placement-auth roles) | Hotel operations | WS7 |
| ODI-07 (scoring calibration beyond adopted contract) | Hotel operations + engineering | WS3 calibration sub-scope |
| ODI-08 (dashboard adopt-or-remove) | Owner at review cluster | WS8 endpoint choice |
| Maintenance-domain lifecycle owner capability / contract realization | Maintenance domain owner (engineering) — declared dependency, sits in ERR-B status | WS7 consumption (before any WS7 build) |

### Wave 1 — Foundation: WS1 (daily generation & identity) and WS2 (lifecycle enforcement) — both gate-free on their APPROVED/EB anchors, first in build order (proposed ordering: WS1–before-WS2 is a planning choice, not source-mandated; no source record ranks them)

### Wave 2 — Workflows: WS3 (inspections — incl. fail-visible mirror leg FIRST, its independent validation precedes any mirror-leg consideration; F5.3/F7.3) and WS4 (staffing/cards — with the [DEL,HKD-23/30]-locked sub-scopes unlocked only by the Wave-0 bundle execution; EB-repair layers proceed on their own authority). Concurrent execution is a planning option only where their sub-scope gates (ODI-07/ODI-11 for WS3; ODI-03/ODI-12/ODI-11 + bundle for WS4) are satisfied.

### Wave 3 — Integrations: WS5 (guest requests — needs bundle HKD-20 + the room-service persistence contract design-verification), WS6 (linen — needs bundle HKD-21 + ODI-06/09 + the ERR-E mapping), WS7 (blocks consumption — needs bundle HKD-04 + the maintenance-domain lifecycle/contract existence + ODI-05). Parallelism only a future planning option where stated dependencies permit.

### Wave 4 — Surfaces: WS8 (dashboards/UI truthfulness — needs ODI-08; underlying data-truth from Wave 1–3 where dashboards read those layers) and WS10 (stack retirement executions — only inside their own later-phase gates per HKD-28/F1.10; underlying data truth zero-dependence). Nothing here executes in Phase 4.

### Independent track — WS9 (authorization): activation ONLY after the OQ-12 sign-off. It is NOT in the main dependency chain and MUST NOT block unrelated work (mandate §4.2); no other wave waits for WS9, and WS9 does not wait for any wave except the sign-off.

**Dependency/sequencing caveats.** The Wave grouping itself is mandate-phase planning structure, not an FDS-recorded obligation; where a dependency or ordering assumption is not supported by the source records, it is marked as **proposed** (WS1/WS2 rank-order; W2-pair parallelism; W4's zero-dependence claim) pending owner or engineering confirmation, per mandate §5.3.

---

## 13. Global verification-integrity statement

All verification in this plan was documentation inspection and read-only source inspection (sections `00` §2–§4). No implementation test, migration, schema operation, runtime check, build, or publication was performed or authorized. Every rule, finding, decision record, open design input, gate, and code touchpoint cited above is traced to the authoritative Phase 1/2/3 records or verified live code this session; anything less is explicitly labeled `UNVERIFIED`/`register-carried` in place, and discrepancies are filed in `00` §6.

**Phase 4 completion ≠ build authorization.** No workstream above is authorized to begin coding by this document; build authorization requires separate owner acts and satisfaction of each workstream's entry conditions and applicable gates as carried forward in `02_DEPENDENCY_AND_GATE_MAP.md`.

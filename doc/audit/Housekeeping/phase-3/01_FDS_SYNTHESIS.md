# XYLO HOUSEKEEPING — FINAL DOMAIN SPECIFICATION (FDS)

Document: 01_FDS_SYNTHESIS.md
Lifecycle step: Phase 3 — Final Domain Specification
Status: COMPLETE — authored 2026-10-20 against the closed Phase 2 ledger
Basis: `docs/audit/Housekeeping/` (Phase 1, locked) + `docs/housekeeping/phase-2/` (locked). Normative labels use the classes verified in `00_PHASE3_SPEC_CONTROL.md` §4. Rule IDs (`F<c>.<n>`) are the traceability keys used by `02_TRACEABILITY.md`.

Reading conventions:
- **[APPROVED]** = owner anchor, binding verbatim · **[EB]** = EVIDENCE_BACKED · **[RETAIN]** = compatibility-mirror disposition · **[DEL]** = delegated recommendation, owner ratification pending · **[PROV]** = provisional pending external fact · **[DEF]** = accepted deferral · **[OPEN]** = open ratification/design input.
- Anchors cite the Phase 2 register (`02 §1 HKD-xx`), closure log (`07 §x`), and the Phase 1 findings register (`HK-xxx`). The FDS states *rules and behavior*; how to build them is out of scope (mandate §1.5).
- Current implemented behavior is cited as evidence of what exists, never as proof of intent or approval.

---

## 1. Scope, successor map, out-of-scope, and reopen guards

### 1.1 Successor-domain map (single-authority rule)

| Capability | Sole owning authority | Basis |
|---|---|---|
| Guest-request lifecycle (hotel requests) | Room-service domain; HK consumes/creates through it | [DEL] HKD-20 (OQ-01 closure; pattern-consistent with HKD-16/HKD-28) |
| Supply-request lifecycle (states, quantities, approval levels) | Inventory / procurement domain; HK orders through it | [APPROVED] HKD-28; [DEF] HKD-29 recorded as that domain's obligation |
| Person-discrepancy detection, occupancy reconciliation | Front Office stack (room-discrepancy domain) is the single authority | [APPROVED] HKD-16; [APPROVED, derived] HKD-17; [EB] HKD-18 |
| Lost-and-found claim affordances | Lost-found module (its own `POST /lost-found/claims` flow) | [EB] HKD-25 |
| OOO / OOS availability-block lifecycle | Engineering-maintenance domain; HK and FO consume; HK may request | [DEL] HKD-04 |
| Room-condition truth | `rooms` row (this module reads/writes only the housekeeping axis) | [EB] HKD-01/31 + [RETAIN] HKD-32 family |

### 1.2 Out-of-scope statements with reopen conditions

| Out of scope | Rule ID | Status class | Reopen condition |
|---|---|---|---|
| Shifts (day/night schedules, property-local time, night-cleaning attribution) | F1.1 | [PROV] HKD-10 | Property evidence of HK shift systems needing dated segregation |
| Laundry service | F1.2 | [PROV] HKD-22 | Proof of an in-house laundry operation → separate capability study (not retroactive HK scope) |
| Workload credits / SLA overdue engine | F1.3 | [DEF] HKD-12 (OQ-07) | Ops actually delivering a formula and a scheduling-optimization requirement; until then `slaMinutes` is display-only |
| True headcount ("people observed in room") | F1.4 | Queue-closure OQ-09 (negative on current evidence) | Evidence of need creates a NEW capability study; never retroactive scope into the approved FO adoption |
| "Teams" grouping | F1.5 | [DEL] HKD-23 (retired) | A real grouping need beyond section assignment would enter the HKD-19 funnel individually |
| HK-facing supplies view | F1.6 | [DEF] OQ-14 | Case-by-case UI-phase item when that work starts |
| Minibar, deep-clean, scheduling-type surface ideas | F1.7 | [APPROVED] HKD-19 (funnel) | Each enters individually with its own justification; blanket presume-required/presume-retire prohibited |
| HK mobile / housekeeper-facing product surface | F1.8 | Explicit not-a-Phase-2-rule disposition (HK-025) | None recorded; a product decision would be an owner act |

### 1.3 Standing rules

- **F1.9 [APPROVED verbatim, HKD-19]** "Case-by-case disposition default for every disconnected capability: no blanket presume-required/presume-retire." The retired list follows, per record: dead `_components` incl. crash-if-render `MinibarSection` → UI-phase retirement candidates (HKD-26 [EB]); duplicate lost-found affordances → delegate (HKD-25); move-room → integrate (HKD-24); dashboard fabricated counters → replace with server-authoritative data (HKD-27); guest-request string protocol → retire in spec (HKD-20); linen → required with real persistence (HKD-21); laundry → excluded by default (HKD-22); teams → retire (HKD-23).
- **F1.10 [RETAIN/EB/DEL mixture per record]** Retirement *decisions* live in the ledger; retirement *execution* belongs to later phases' own approval gates. The FDS marks components/routes/data semantics as retired in the specification and prohibits new work on them; it authorizes no removal today (matches `02 §1` HKD-28 "retirement executes in later phases' own gates").
- **F1.11 [EB, HKD-05 gate-compat + 02 §2 closure audit]** No decision weakens the approve-only re-entry rule; access-control binding of it is subject to OQ-12 (Ch 14).

## 2. Room-state model and vocabulary

### 2.1 Condition vocabulary [EB, HKD-01]

- **F2.1** `rooms.housekeeping_status` carries the authoritative condition vocabulary: **DIRTY / CLEAN / INSPECTED**, plus the legacy defaults **READY** (room-level schema default) and **PICKUP** (task-level concept). Progress/combined states ("cleaning in progress") are task- or display-derived, never room-column values.
- **F2.2 [EB verbatim, HKD-05]** READY is "legacy default equivalence ('accept as cleanable-ready'), never an inspection outcome/outward write target." `READY ≠ INSPECTED` reading-drift is prohibited: no inspection logic writes READY (anchors `room-readiness.spec.ts:43–44`).
- **F2.3 [EB, HKD-02/03 via 07 §7 HK-014]** CLEAN exists as a produced working state: task-completion for cleaning tasks is CLEAN's producer (spec requirement detail under the approved approve-only rule). Approval then produces INSPECTED. Nothing else writes INSPECTED; nothing writes READY.
- **F2.4 [EB, HK-021]** Task-status vocabulary is the implemented set `pending | in-progress | completed | inspected | rejected | cancelled`; priority vocabulary fixed per the implemented default. External UPPER-case constant families are display artifacts; constants-file casing/priority alignment is implementation-stage work, not schema work (HKD-01 consequence recorded in `02 §1`).

### 2.2 Axis separation [EB, HKD-31; DEL, HKD-06]

- **F2.5** The operational availability axis (`rooms.room_status` values incl. VACANT/OCCUPIED/AVAILABLE-family and OOO/OOS-adjacent literals) is owned by FO/reservations. HK is strictly read-only on it. The readiness gate composes both axes (condition × availability × block), documented, never merged.
- **F2.6 [DEL verbatim-scope, HKD-06]** `rooms.front_office_status` is not authoritative in HK; **zero HK writers** (zero-writer = policy). The column is demoted to compatibility; the FO namespace owns any future concept decision. The FO comparator's HK-state read must be re-scoped (see Ch 7, F7.2 same-wave rule) before the read-fallback path is removed (removal = implementation-stage).

### 2.3 Transitions and gates

- **F2.7 [EB, HK-026 + approved lifecycle HKD-02/03/09]** The task transition matrix enforced nowhere today is here defined and specified as *enforced* (enforcement realized in implementation phases): no any-to-any transitions; `inspected → pending`/`cancelled` prohibited; the module's own supply-request service demonstrates the codebase's enforcement standard.
- **F2.8 [EB, HKD-05 + HK-027 + 02 §5 row 4]** Canonical readiness acceptance for the check-in pipeline = the implemented gate set {CLEAN, INSPECTED, legacy-default READY}; INSPECTED is sufficient everywhere. The scheduled-room-move consumer's stricter set (rejects READY) is recorded as a consumer choice (ODI-12); the parallel walk-in/legacy assignment path currently permitting DIRTY must be aligned to the canonical gate — spec requirement, later implementation (HK-016 disposition). Reservations/group-alotment sellability ignores the condition axis by design (correct separation).
- **F2.9 Failure cases recognized in the spec:** absence of a status field means *not ready* (FO-side "never fabricates readiness" posture adopted; same rule as F13.1); a legacy READY-room entering the gate passes *as legacy default*, not by inspection logic; no fabricated CLEAN display (Ch 13).

### 2.4 Mirror-family implication [RETAIN, HKD-32; EB, HKD-07/08]

- **F2.10** No HK rule in this specification may introduce or rely on the legacy `room_status` mirror, `room_status_history`, `room_status_config`, or `dnd_status` as domain facts. See Ch 7 for the full retention contract.

## 3. Business date and daily generation

### 3.1 Generation semantics [APPROVED, HKD-09; APPROVED-derived, HKD-13]

- **F3.1 [APPROVED verbatim, HKD-09]** "**Ensure-missing generation:** daily generation creates missing daily work only; completed/in-progress/inspected tasks never reset; duplicates = no-op; no mid-run abort on key conflicts; trigger authority = HK supervisor (manual only; automation = separate enhancement approval)."
- **F3.2 [APPROVED-derived verbatim, HKD-13]** "Task-preservation + concurrency invariants: never reset progressed work; transactional batch; skip-on-conflict; audit reference preserved."
- **F3.3 [EB, HKD-11 + ODI-01]** "Daily" requires an effective business-date attribution key. Nearest-evidence family: `(hotel_id, effective_business_date, room_number, task_type)`; exact composition is ODI-01; the key is system-attributed and collision-safe. Idempotency: repeated generation for the same effective date is a no-op for existing keys.
- **F3.4 [EB, HK-005]** Client-minted identifiers are prohibited as persisted keys/business ids in spec; the server allocates identities (the `HK-${Date.now()}` / `TASK-${Date.now()}` millisecond-collision family specified as illegal behavior).
- **F3.5 [EB, HKD-14, anchors `check-out.handler.ts:470–481`]** Checkout marks the room DIRTY (this behavior stands). Generation creates a checkout-adjacent task only for DIRTY rooms lacking matching active non-guest work; no duplicate assignment. Departure-day vs stayover distinction is by trigger source; naming/merge specifics = ODI-02.
- **F3.6 [EB, HK-009 resolution via F3.1/F3.3]** The unique-key collision aborting generation mid-loop (HK-009) is dead in spec: system-attributed ids + no mid-run abort on key conflicts.

### 3.2 Failure cases specified

- **F3.7** Generation under clock/date skew: the effective business date is property-attributed (per-date precedents: `room_turndown`, `room_discrepancies` uniqueness); a skewed run cannot mutate progressed work (F3.1/F3.2 bound the outcome set), so skew degrades to no-op or to the correct-ticket set — destructive outcomes are impossible by construction.
- **F3.8** The policy-violating destructive reset (current implemented behavior, HK-004) is *detectably illegal* under this spec: it erases progressed work and writes no audit; any future behavior equivalent to it violates F3.1/F3.2 and the audit rules of Ch 4.

## 4. Task lifecycle and transition semantics

### 4.1 Ownership and transitions

- **F4.1 [DEL, HKD-30]** Per-room work ownership is authoritative on `housekeeping_tasks.assigned_to`; `hk_room_assignments` are day-plan cards (identity/handover data), never work-state storage. Card rooms-array JSON is a plan (shape = ODI-03).
- **F4.2 [EB, F2.7 continuation; HK-026]** Enforced status matrix for tasks: `pending → in-progress → completed → inspected`; `rejected` cycle per Ch 5; `cancelled` only pre-completion; approved-only transitions both directions prohibited; audited transitions labeled with the acting identity.
- **F4.3 [EB via 07 §7 HK-014/F2.3]** Task completion of a cleaning task produces the CLEAN room state (the intermediate between DIRTY and post-approval INSPECTED); completion alone never claims *entered-service readiness* — that is solely approval (HKD-02).
- **F4.4 [EB, HKD-27 + HK-018]** All state writes are fail-loud, including configuration writes (`hk_sla_config`, `hk_status_flow_config`): a failed persistence reports failure; returning success without persistence is prohibited; per-instance uninvalidated caching conflicts with truthfulness and is specified out (refresh contract specified in implementation phases).

### 4.2 Attribution, handover, linkage

- **F4.5 [EB/APPROVED-adjoined, HKD-03/HK-008]** Every mutation that names a human actor records the **authenticated** identity (userId/name from request context, never client-supplied fields), the reason where the act is discretionary, and timestamps. The approved rejection rule makes this binding for inspections (Ch 5); the same rule is adopted for all HK mutations as engineering standard.
- **F4.6 [EB, HKD-15]** Handover: no automatic transfer of unfinished work; carry-forward with original assignee; reassignment is an explicit, audited human act. Reopen condition documented (an ops auto-bucket rule could arrive later).
- **F4.7 [DEL, HKD-20 + HK-030]** The guest-request string protocol (`Guest:` note encoding, regex-parsed linkage, note-sniffing in generation/queries, fire-and-forget request completion, direct task PATCH from GuestRequests) is retired in spec. Until real persistence exists through the room-service contract, HK must not present workable fake request flows; any linkage goes through the room-service authority (F10.1).
- **F4.8 [EB, HKD-24]** Task move-room is a task-level integration for later phases (service method exists unrouted; UI call dead is part of the retirement list); invariant: a move cannot violate HKD-02 (re-entry/condition rules) or OOO/OOS reads (Ch 9).

## 5. Inspections and room re-entry (core chapter)

### 5.1 Approved anchor and rejection lifecycle

- **F5.1 [APPROVED verbatim, HKD-02]** "**Approve-only re-entry:** inspection approval is the sole authorized DIRTY→service transition; nobody else declares readiness; swallowed-failure design declared unacceptable (fix owned by later phases)."
- **F5.2 [EB, HKD-03 + verified reject path (`07 §2`, `03 §9`)]** Rejection: (i) leaves the room DIRTY-class/unsaleable (room condition is *never written* by rejection — verified behavior adopted as the rule); (ii) **creates/targets a re-clean task — auto-create is a Phase 3 spec requirement** (today's no-auto-reclean default was verified; the spec levels it up as required); (iii) audit-logs the **authenticated** inspector identity, reason, room, and outcome (client-supplied inspector fields are prohibited; HK-008's blank/spoofable trail is dead in spec).
- **F5.3 [APPROVED-binding + EB]** The approve leg must be fail-visible: the swallowed mirror write (the `.catch(()=>{}` family, HK-006/HK-008) is prohibited behavior with required error surfacing. Until the mirror leg is re-terminated (Ch 7), its failure aborts approval with a surfaced error rather than reporting success.

### 5.2 Invariants, edge cases, scoring

- **F5.4 [EB, `03 §5` invariant]** A room must never be saleable while an inspection is REJECTED/pending-resolution.
- **F5.5 [EB, `03 §5`]** Repeated-rejection ping-pong: no auto-escalation is designed; the escalation owner is the HK supervisor (documented consequence of the rejected design space).
- **F5.6 [EB, HKD-31 interlock + `03 §5`]** Reject-on-occupied-room: HK writes only its condition axis; occupancy truth is FO/reservations-owned; the FO side controls entry (check-in gate). The reject path never corrupts, writes, or derives occupancy; reconciliation is by axis composition (F2.5), not overrides.
- **F5.7 [OPEN, ODI-07]** `room_inspections` scoring: current implemented semantics (fail row score 0; approve row hard-coded 100) are adopted as the current contract and recorded; score calibration is an open design input (nothing invented here).
- **F5.8 Specified failure cases:** concurrent approve/reject of one inspection is serialized by the task state machine (first committed transition wins; the loser surfaces a conflict, never both-write); approval outside an inspection workflow is illegal (no bare room-status writer); every approval/rejection is durable with audit reference preserved (HKD-13 consequence for the inspection edge).

## 6. Staffing, assignments, and day-plan

- **F6.1 [DEL, HKD-23]** Attendants + sections are the single staffing truth; "teams" is retired (no team table ever existed); the dead `/housekeeping/teams` affordance is marked for later UI-phase removal. Sections assignment is per hotel+employee (implemented uniqueness adopted).
- **F6.2 [DEL, HKD-23]** Supervisor distinction lives on assignment cards only (the implemented `type: supervisor` cards), not as a role column on attendants. The designation of who inspects is part of the OQ-12 security chapter (Ch 14, ODI-10).
- **F6.3 [DEL, HKD-30]** Day-plan cards exist for handover/planning; their rooms-array is a plan, never work-state. Shape: ODI-03.
- **F6.4 [DEL/EB, HKD-30 + HK-011]** `reassignInspection` writes supervisor ids only on supervisor-role cards, never onto attendants' task attribution; task `assigned_to` remains the attendant's work ownership (re-spec of HK-011; both current corrupting behaviors specified out).
- **F6.5 [DEL, HKD-23 consequence + EB HK-024]** An attendant **provisioning write path is a Phase 3 spec requirement** (creating/managing attendant records tied to employees/sections in product). Today's zero-writer state is a registered gap the spec closes on paper; realization belongs to the implementation phases.
- **F6.6 [EB, HKD-12-related, HK-012]** Staff identity joins must use the FK-backed key (`housekeeping_attendants.employee_id → employees.id`); the current join against the business-code column is specified out. Staff provisioning data requirements follow F6.5.

## 7. Room-status authority vs legacy mirrors

- **F7.1 [RETAIN, HKD-32 + EB, HK-013]** The `rooms` row is the truth for room condition. The legacy `room_status` table (+ `room_status_history`, `room_status_config`, `dnd_status`) is retained as a **compatibility mirror**: zero consumers were verified statically and at runtime catalogs/data-state in the safe environment (`07 §3`, §10); no Phase 2/3 removal is authorized; removal safety is *not claimed*.
- **F7.2 [DEL+APPROVED-derived, HKD-06/17/18; `04 §3` same-wave rule]** The FO comparator's vacant-state re-anchoring to `housekeeping_status` (within the FO-owned flow) is specified **in the same specification wave** as the HKD-06 demotion; Phase 4+ implements both before the read-fallback path removal. Until then the FO comparator's behavior is unaffected by this ledger. The re-anchored vacant-state set definition is ODI-04 (FO co-sign constraint carried).
- **F7.3 [RETAIN]** No new consumers of the mirror family may ever be introduced; any state a feature needs must come from `rooms`/domain tables. The sole existing writer is the HK approve leg, now fail-visible (F5.3), scheduled for re-termination under the approved rule.
- **F7.4 [RETAIN, `07 §10.4` staged removal prerequisites]** Removal itself is gated to a later DB phase with four prerequisites: (1) HK approve mirror write retired/re-terminated; (2) the FO `room_status_history` read (`get-room-detail.query.handler.ts:72`) re-scoped or re-sourced; (3) the RLS-enabled-with-zero-policies state consciously resolved (latent trap recorded — `07 §10.2`); (4) divergence re-check on a populated safe copy. This FDS commits only the proposal, not execution.
- **F7.5 [EB, HKD-07/08]** `room_status_config` (legacy display order) and `dnd_status` (mirror column) have zero consumers: **excluded from the HK specification** entirely; listed in the mirror-retirement evidence.

## 8. Discrepancy and occupancy

- **F8.1 [APPROVED verbatim, HKD-16]** "**Person discrepancy: Front Office stack is the single authority (`room_discrepancies`); HK-tagged CQRS stack dispositionally retired; HK UI consumes FO data if/when needed.**"
- **F8.2 [APPROVED-derived verbatim, HKD-17]** "Occupancy authority: FO in-house reservations + `room_status` are the occupancy facts; HK never writes occupancy truth; comparator vacant-states re-anchored to `housekeeping_status`."
- **F8.3 [EB, HKD-18, verified-adopted FO semantics]** Record lifecycle: OPEN on detection with business-date uniqueness; RESOLVED requires a user; reopen-on-recompute semantics retained; HK supervisors **consume**, do not resolve.
- **F8.4 [DEL/EB, HKD-16 family; HK-003]** The HK discrepancy stack is retired in spec: the missing-table `person_discrepancies` surface, the `GET /housekeeping/discrepancies` 404 call, `/mobile/discrepancies` read/write family and person-count writes get no verb adoption and no product behavior; nothing in this spec defines them (FO co-sign constraint on the re-scope wave applies — `07 §4` HKD-06 line).
- **F8.5 [queue closure OQ-09]** The FO reconciliation authority detects room-level occupancy mismatches, not people counts; true-headcount remains outside the approved adoption (guardrail restated; F1.4).

## 9. OOO/OOS availability blocks

- **F9.1 [DEL, HKD-04]** The engineering-maintenance module owns the OOO/OOS lifecycle end-to-end (placement, release); HK and FO are consumers; HK may **request** a block but never place or release one on its own authority.
- **F9.2 [DEL+EB, HKD-04 + HKD-31]** Block state is consumed through the availability axis rules (F2.5): HK task generation and assignment decisions respect current block state (blocked rooms are not worked); HK never writes the axis values.
- **F9.3 [OPEN/DEL, ODI-05]** Placement-authorization roles (who authorizes each cause: dirt-caused vs fixture vs renovation) are an operations process question — the contract names the owner; the role matrix is the ops input.
- **F9.4 [EB, HK-019/HK-20 boundary]** Literal fragmentation across `{OUT_OF_ORDER|OOO|OOOS|OOS|OUT_OF_SERVICE}` readers is a vocabulary hazard: block literals normalize according to the maintenance-domain contract; the FO `RoomStatus` value-object `'OOO'` acceptance is an FO-domain boundary note, not an HK rule.

## 10. Guest requests

- **F10.1 [DEL, HKD-20 register text]** "Guest requests: room-service authoritative; HK string protocol retired in spec; HK requests via FO-adjacent live routes; persistence+audit required end-to-end."
- **F10.2 [DEL/EB, HKD-20 + HK-030]** Consequences: the note-protocol retirement in Ch 4 applies here; a linked task must never silently complete its source request (fire-and-forget prohibited); until the room-service persistence lands, the HK UI must not present workable fake request workflows (HK-001 request endpoints part of the retirement list).
- **F10.3 [DEF, HKD-12 family]** SLA/display expectations for requests are display-only until a formula is delivered; no overdue engine is specified.

## 11. Linen and inventory adjacency

- **F11.1 [DEL, HKD-21]** Linen management is a **required capability with real persistence** — the server already reads real `linen_inventory` fields (`quantity_on_hand`/`par_level`/`unit_cost`); the issue/return/damage flows are specified at contract level with audit end-to-end (ODI-06 par-location alignment and ODI-09 operational details are ops inputs).
- **F11.2 [DEL/EB, HKD-21 + HKD-27 norm]** The synthesized client-side refill loop (HK-002 fabricated movements) and the synthesized fixed-ratio location splits (HK-035) are prohibited: linen numbers shown anywhere must come from persisted real data. Missing data displays as missing — never fabricated.
- **F11.3 [PROV, HKD-22]** Laundry is excluded from the HK spec (in-house vs outsourced unresolved); proof of an in-house operation triggers a separate capability study.
- **F11.4 [APPROVED verbatim, HKD-28 + DEF, HKD-29]** "Supply requests: inventory/procurement domain owns the lifecycle; HK orders through it; HK-tagged stack dispositionally retired (retirement executes in later phases' own gates)." State/quantity/approval-level policies are the inventory domain's concern (HKD-29 recorded as inventory-spec input). The HK supply stack — including the dead approval transition (HK-037, whose approve endpoint can only 400) — is retired; no dual lifecycle is specified.
- **F11.5 [DEF, OQ-14]** No HK-facing supplies view is a Phase 3 obligation; it arises case-by-case at the UI phase.

## 12. Dashboards, reads, and data truthfulness

- **F12.1 [EB, HKD-27]** Rendered operational counters (requests, inspections, tasks, dirty/clean/inspected/rejected and similar) must be **server-authoritative**: the implemented `GET /housekeeping/dashboard` counters were verified as server-computed, which is the ready candidate data source (ODI-08 adoption-or-removal pending ratification at review; HK-034).
- **F12.2 [EB, HKD-27 + OQ-13 closure]** Client-synthesized/fabricated counters are prohibited — the OQ check found no ops case for fabricated counts; `DashboardSection`-style pseudo-task synthesis, invented SLA values, "AI Route Optimization Active" banners, fake spinners, dead Reassign buttons are all specified out (HK-022 register detail).
- **F12.3 [EB, HK-033]** The audits/history tab refresh contract (stale-after-approve until re-entry) is a UI-phase consumption rule — data-display correctness follows the truthfulness norm.
- **F12.4 [EB, HKD-24/25/26]** Capability dispositions recorded for the UI phases: move-room integrated as task-level operation (F4.8); lost-found claim affordances delegated (F1.9); six dead `_components` (incl. crash-if-render `MinibarSection`) are retirement candidates — no silent resurrection (F1.9 anchors).

## 13. UI contract

- **F13.1 [EB, HKD-06/31 + HKD-27 norm; HK-028]** The frontend may not synthesize conditions: a missing `housekeeping_status` renders as *not ready/unknown*, never as `'CLEAN'` — the FO-side "never fabricates readiness" posture is adopted module-wide.
- **F13.2 [EB, HKD-06/31/01; HK-029]** Rack/room grids must separate axes: housekeeping condition, occupancy (FO-owned), and block status are distinct data sources; the mixed `room.status`/`housekeepingStatus` filter bug and FO rack condition derivations are FO-owned defect sites (boundary noted), whose correct behavior this contract defines.
- **F13.3 [EB/DEL mixture per record; HK-001]** Endpoints that do not exist are never called; affordances without a backend are removed in later UI phases: every dead URL family in HK-001's Annex-A right column (linen writes, laundry, supplies writes, requests, teams, move-room, claim) maps to its retire/integrate/delegate disposition in Ch 10/11/1/6/25; linen screens read real data (F11.1/F11.2).
- **F13.4 [explicit not-a-rule, HK-025]** No mobile or housekeeper-facing product surface is specified by this FDS (the static mock and the unused mobile endpoints are part of the retired-stack family in F8.4).

## 14. Security and authorization [PROVISIONAL chapter — OPEN RATIFICATION]

> Activation gate: this chapter is **PROVISIONAL — OPEN RATIFICATION**. Its approval-condition rules become fully normative only upon the explicit OQ-12 sign-off (`07 §9.4`); until then they are the ratification-ready intended state, clearly separated from the verified current state. No sign-off occurred during this session; none is fabricated (Mandatory Corrections #5/#7).

- **F14.1 [verified current state; 07 §9.1]** Enforcement chain: JWT + property-scope + multi-tenancy globals, plus two metadata-driven permission guards (common `@Permissions`, platform `@Permission`) and an unused `RolesGuard`; every permission/role guard passes any route lacking its decorator.
- **F14.2 [verified current state; 07 §9.1]** All 62 Housekeeping routes carry no role/permission decorators: every HK operation — including inspection approve/reject, which binds the approved HKD-02 rule — is open to any authenticated tenant-scoped user. This is implemented fact, not intent.
- **F14.3 [verified current state; 07 §9.1]** Registry: HK department roles (`HOUSEKEEPING_MANAGER`, `HOUSEKEEPING_STAFF`) receive only `command-center:*`/`common:*` grants; the `housekeeping:*` codes referenced by command-center surfaces are consumed-but-never-defined-or-granted; no `@Roles(` decorator is used anywhere in the API.
- **F14.4 [INTENDED — OPEN RATIFICATION, matrix from `07 §9.2`]** Intended role→action policy (ratification-ready): SUPER_ADMIN/ADMIN bypass as today; HOUSEKEEPING_MANAGER: generate daily run, approve/reject inspections, reassign, config read/write, dashboard/audit reads, linen ops; HOUSEKEEPING_STAFF: own task start/complete + own plan visibility; FRONT_DESK roles: no HK route calls (FO stack covers consumption); other department roles: none; inspector-role determination (manager-only vs new HOUSEKEEPING_SUPERVISOR) is an open determination inside the sign-off.
- **F14.5 [OPEN determination]** Adoption items requiring the explicit sign-off: (a) `housekeeping:*` registry definitions/grants; (b) the route decorator plan for the 62 routes; (c) guard-system choice — common `@Permissions` vs platform `@Permission`, whose ADMIN-bypass semantics differ; (d) inspector-role determination.
- **F14.6 [DEL, HKD-16-family]** Retired stacks (HK discrepancy/mobile family) need no authorization verb adoption; their routes remain pending stack retirement (`07 §9.3` gap 5).

## 15. Integration boundaries and cross-module contracts

| Boundary | Producer/owner | HK role | Prohibition |
|---|---|---|---|
| FO stack (discrepancies, room detail, rack, due-outs) | FO domain | Consumer (permission-gated on FO side) | HK never writes occupancy truth or the availability axis (F2.5/F8.2) |
| Room-service (guest requests) | Room-service domain | Consumer/creator through its live routes (F10.1) | No string protocol (F4.7); no dual request lifecycle |
| Engineering-maintenance (OOO/OOS) | Maintenance domain | Consumer; may request a block (F9.1) | HK never places/releases blocks on the axis |
| Inventory/procurement (supplies) | Inventory domain | Orders through it (F11.4) | No HK-owned supply lifecycle; no HK state/quantity/approval policy (HKD-29) |
| Reservations room-move contract (F-25) | Reservations writes `rooms` row columns, not the mirror table (verified; `07 §3`/`03 §10.1`) | Consumer of resulting state | HK does not dispute or re-mirror the write |
| Lost-found | Lost-found module | Delegated (F1.9) | No duplicate affordances (HKD-25) |
| Notification/message-log adjacency | Messaging infrastructure | Consumer/producer of notifications | Server-minted message ids required (HK-005 millisecond-collision family — `notif-${Date.now()}`-style ids are specified out); authenticated identity everywhere (F4.5) |

- **F15.1 [EB, HK-017]** Schema-governance boundary adopted as spec rule: no run-time DDL; raw-SQL discipline must remain tenant-true by explicit scoping; `ensureAuditLogTable`-style runtime creation and unscoped `$queryRawUnsafe` prose are implementation-phase prohibitions with a named violation class — the design may rely only on migrated schema.
- **F15.2 [EB, per-boundary rows above]** Every cross-module contract names its failure behavior: producer outages surface as consumer errors (never as silently degrading fabrications); the countable-attribution rule (F4.5) binds all contracts that carry human actions.

---

*End of FDS. Chapters 1–15 complete; rule IDs `F<c>.<n>` are the traceability keys used by `02_TRACEABILITY.md`. Authority classes follow `00 §4`; the security chapter's activation gate is F14 (OQ-12 sign-off).*

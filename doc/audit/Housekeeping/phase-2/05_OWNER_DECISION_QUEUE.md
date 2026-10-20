# XYLO HOUSEKEEPING — PHASE 2 OWNER DECISION QUEUE

Document: 05_OWNER_DECISION_QUEUE.md
Lifecycle step: Phase 2 — Business Rules Decision
Status: COMPLETE (queue open — owner session ended 2026-10-20 with five decisions APPROVED; remaining items below)
Date: 2026-10-20

Reading note: items are grouped to avoid asking the owner the same question twice. Priorities: **P1 = blocks Phase 3 specification chapters**; **P2 = blocks individual capabilities or implementation timing**; **P3 = low urgency / evolution items**. Each item states the issue in plain language, the viable choices, the operational consequences, an evidence-backed recommendation where one exists, and any extra information required. The 2026-10-20 owner session has already resolved five anchors (HKD-02, 09/13, 16/17, 19, 28) — they are not re-asked.

---

## Part A — P1: owner answers that gate Phase 3 chapters

### OQ-01 — Guest-request workflow (HKD-20) [P1]
- **Issue.** Both hotel-requests tabs write requests that never persist (frontend-only echo; the write endpoints do not exist). The existing live request module is room-service. Keep Housekeeping requests, route them through room-service, or retire the HK tab's write affordance?
- **Choices.** (a) Room-service domain owns request lifecycle; HK consumes/creates through it (recommended); (b) HK owns a request lifecycle of its own (new domain work; duplicates room-service); (c) Retire requests from the HK UI (operations requests stay in room-service alone).
- **Consequences.** (a) one request system, HK screen becomes a real workflow once wired; (b) two request lifecycles and two audit trails to keep consistent; (c) removes a currently-fake screen that operators may believe works.
- **Recommendation.** (a), per the single-authority norm the owner set for discrepancies (HKD-16) and supplies (HKD-28): same pattern, same rationale.
- **Extra info needed.** None beyond choice; SLA/display expectations are a later config item (HKD-12 family).

### OQ-03 — Who may declare a room OUT OF ORDER / OUT OF SERVICE (HKD-04, interacts HKD-31) [P1]
- **Issue.** No module anywhere in the API writes the availability blocks, although every screen shows them. If Housekeeping owns OOO placement, a cleaning constraint becomes a room-availability fact written by HK; if maintenance owns it, HK can only request.
- **Choices.** (a) Engineering-maintenance module owns both OOO and OOS lifecycle, with HK and FO as consumers (recommended); (b) HK owns OOO placement (cleaning-caused) and FO owns OOS (rooms unusable for service); (c) FO owns both.
- **Consequences.** (a) one owner of the availability axis, consistent with HK being read-only on availability (HKD-31 recommendation); (b) needs a tight two-writer matrix and an HK→FO message path that today does not exist; (c) removes HK's natural interest (blocked-because-dirty) from its own domain.
- **Recommendation.** (a).
- **Extra info needed.** Confirm the property processes that actually create blocked rooms (dirt-caused vs broken-fixture vs renovation) and who authorizes each — this is an operations question.
- **Note.** Choosing (b) would contradict the PROPOSED HKD-31 read-only rule; if you choose (b), HKD-31 must be amended before Phase 3 (see 04 §3).

### OQ-06 — Shifts: in or out of scope (HKD-10) [P1]
- **Issue.** The legacy store has shift-shaped strings, but no shift concept exists anywhere in the domain: no model fields, no boundaries, no property-local time handling. Adopting a shift model means defining, per property, shift names/hours and what belongs to each shift; deferring keeps the domain as a single daily workday model.
- **Choices.** (a) Out of scope for the Housekeeping domain spec — daily business-date model only (recommended); (b) In scope — shifts become first-class assignment/task attribution now.
- **Consequences.** (a) later addition is a normal change; no speculative semantics are carried; (b) Phase 3 must design shift schedules, handover rules (HKD-15 view), night-cleaning attribution, and likely a scheduling surface.
- **Recommendation.** (a), unless properties actually run HK shift systems that need tracking today — that evidence does not exist in this repository.
- **Extra info needed.** Whether any property operates an HK night/PM-shift needing dated segregation, and whether staffing hourly coverage rules will be tracked in this product at all.

### OQ-10 — Linen & laundry disposition (HKD-21, HKD-22) [P1 for linen chapter]
- **Issue.** All linen/laundry backend flows are unreachable, and the linen "issue/return" loop fabricates data on screen only (HK-002). Question 1: is persisted linen management (issue/return/damage with real counts) an approved capability? Question 2: is laundry run in-house (full lifecycle needed) or outsourced (minimum contract/record tracking)?
- **Choices.** Linen: (a) required — build real flows in later implementation phases (recommended; seed par data already exists); (b) minimum read-only display of par levels (the synthesized splits are wrong and would need the real model first); (c) retire from HK (linen belongs to inventory).
- Laundry: (a) in-house — full lifecycle; (b) outsourced — record/vendor model minimum; (c) retire from HK scope.
- **Consequences.** Without the linen option (a), supervisors manage imaginary stock (the recorded HK-002 harm); the laundry decision defines whether an entire sub-domain is specified in Phase 3.
- **Recommendation.** Linen: (a). Laundry: owner decision — no implementation evidence exists to recommend in-house vs outsourced (blocked on property operations).
- **Extra info needed.** Which property processes exist for linen (per-room pars vs per-location topping-up) and the laundry operational model.
- **Grouped.** With OQ-11 (staffing) and OQ-07 (workload) it determines the size of the ops-provisioning chapter.

## Part B — P2: capability and policy confirmations

### OQ-11 — Retire "teams" in favor of attendants + sections (HKD-23) [P2]
- **Issue.** A /housekeeping/teams concept is called but does not exist server-side; the real staffing model is attendants assigned to sections (unique per hotel+employee) with supervisor distinction carried on assignment cards. Keep teams, or make attendants+sections the single staffing truth?
- **Choices.** (a) Retire teams; use attendants+sections only (recommended — one staffing model); (b) Build a teams entity (grouping attendants) for coverage/scheduling purposes.
- **Consequences.** (a) deletes a fake capability from the backlog; (b) requires a real grouping model and Phase 3 definition before anything can be built.
- **Extra info needed.** Whether some property staffing practice genuinely groups attendants into formal teams for accountability beyond section assignment.

### OQ-12 — Housekeeping authorization role model (HKD-02/HKD-03 roles) [P2; security owner]
- **Issue.** Today all 62 HK routes accept any authenticated tenant user (Phase 1 HK-007). The approved lifecycle implies roles: attendants (assign/complete), supervisors/inspectors (approve/reject/reassign), front desk (consume only).
- **Choices.** (a) Restrict HK mutations to named permission scopes tied to the existing registry seeds (recommended); (b) Ad hoc role checks without registry scopes. Owner + security must ratify which existing roles map to which HK verbs.
- **Consequences.** Without (a), the approved approve-only policy (HKD-02) is trivially bypassable by any user; (a) is consistent with the FO/front-office permission-seed pattern in the codebase.
- **Extra info needed.** Owner/security sign-off on the role→verb map; engineering maps it onto the registry in Phase 3 text.

### OQ-08 — Handover policy light-confirm (HKD-15) [P2]
- **Issue.** PROPOSED rule: unfinished tasks carry forward with their original assignee; no automatic transfer; reassignment is an explicit, audited human act. Confirm, or state the desired auto-bucket rule.
- **Choices.** (a) Confirm carry-forward + manual reassignment only (recommended default); (b) Auto-assign unfinished tasks to the next shift/attendant per some rule (requires the ops rule to be given now).
- **Consequences.** (b) silently retargets staff attribution — needs a formula (OQ-07 family) and audit semantics.

### OQ-07 — Workload/credit policy inputs (HKD-12) [P2 informational]
- **Issue.** No workload formula exists anywhere; SLA config is display-only. If credits ever become real, the formula (rooms per attendant, credit values, floors/ceilings) is an operations decision; until then the spec treats credits as absent and SLA minutes as display-only.
- **Choices.** Provide formula inputs now, or accept absence for Phase 3 (recommend accepting absence — nothing in evidence requires credits).
- **Consequences.** Only affects a future scheduling-optimization capability, which today has no approved requirement.

### OQ-09 — Headcount requirement (evidence request, HKD-16 guardrail) [P2]
- **Issue.** The approved authority (FO reconciliation) detects room-level occupancy mismatches, not people counts. If any property or legal process requires true headcount ("people observed in room"), it is a NEW capability requiring its own evidence and approval — not covered by the approved FO adoption.
- **Action.** Owner/ops confirm yes/no. If yes: a separate capability study (sources: room transfer registers, guest-service observations) is queued for a future phase; nothing is added to this ledger's scope.

## Part C — P3 / later-phase confirmations

### OQ-02 — Standing case-by-case review batch (HKD-19 channel) [P3]
- Covers the small groups not pulled out above, per the approved case-by-case default: dead components (HKD-26: six dead `_components`, incl. crash-if-render `MinibarSection` → retirement list at the later UI phase), duplicate lost-found affordances (HKD-25 → delegate to lost-found module), move-room integration (HKD-24 → integrate small), dashboard fabricated counters (HKD-27 → server-authoritative replacement via the unused `GET /housekeeping/dashboard` or removal; OQ-13 is its low-priority ops-case check), minibar/deep-clean/scheduling-type ideas (HKD-26 adjacent → each needs its own justification before any build).
- No single decision is required; the queue expects each to surface with its own evidence when its implementation phase arrives.

### OQ-05, OQ-14, OQ-15 — Ratifications bundled into the ledger review session (P2 event, not a questions list)
- **OQ-05** HKD-06 confirmation with FO (front_office_status demotion policy, incl. comparator re-scope order constraint from 04 §3).
- **OQ-14** Whether an HK-facing view of supplies should be scheduled at all after HKD-28 (new capability for the UI phases; also OQ-13-family).
- **OQ-15** HKD-30 ratification (assignment representation) — data-contract stakes make this owner-visible; no alternative is pending beyond the documented rejected options.

### OQ-20 — Mirror census (evidence work, engineering-led; owner informed) [P3 blocker-closure]
- **Evidence requirement behind HKD-32 (BLOCKED_BY_EVIDENCE).** Full consumer census of the `room_status` mirror table, `room_status_history`, and `room_status_config` (static + runtime data check in an approved safe test environment; no production data). Output: either a retirement proposal for a later DB phase or an indefinite-compatibility disposition. Until then no mirror writes may be added or removed; the interim compatibility-mirror policy stands.

---

## Queue summary

| Priority | Items | Decisions gated |
|---|---|---|
| P1 (owner answers gating Phase 3 chapters) | OQ-01, OQ-03, OQ-06, OQ-10 | HKD-20, HKD-04, HKD-10, HKD-21/22 |
| P2 (capability/policy confirmations) | OQ-11, OQ-12, OQ-08, OQ-07, OQ-09 | HKD-23; HKD-02/03 roles; HKD-15 confirm; HKD-12 evidence; HKD-16 guardrail |
| Ledger-review ratifications (P2 event) | OQ-05, OQ-14, OQ-15 (+13 PROPOSED confirmations incl. HKD-01, 03, 05, 06, 11, 14, 18, 24–27, 30, 31) | — |
| P3 / evidence work | OQ-02, OQ-13, OQ-20 | HKD-27 ops case; HKD-32 census |

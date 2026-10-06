# Phase 6 — 02B Owner Decision Resolution

**Artifact type:** decision-preparation artifact (Phase 6).
**Status:** PREPARED — awaiting explicit owner resolution. **No decision has been made by this document.**
**Companions:** `01_FORENSIC_AUDIT.md`, `02_DECISION_RESOLUTION_ANALYSIS.md`, `02A_OPEN_DECISION_EVIDENCE_RESOLUTION.md`.

**Standing prohibitions (binding on everything below):**

No production code, schema, migration, DB data, test, reconciliation run, cutover, flag flip, legacy deletion, or legacy writer retirement.
No Implementation Plan, no implementation tasks, no forensic audit, no decision-analysis re-run, no evidence-resolution re-run.
No new business rules. **No option silently chosen on behalf of the owner.**

This document exists for exactly one purpose: present the four remaining owner decisions from `02A` in a form the owner can answer directly.

---

## 0. Scope of this document

`02A_OPEN_DECISION_EVIDENCE_RESOLUTION.md` closed 7 of 30 open items, deferred 7, gated 3, left 8 as evidence-required, and reduced the owner queue to **4**. Its verdict: `READY FOR IMPLEMENTATION PLANNING WITH GATED ITEMS`.

| Decision | `02A` ID | Prior ID | Subject |
|---|---|---|---|
| 1 | OI-03 | BD-6-03 | `POST /rates/engine/modify` legacy writer EOL / retirement timing |
| 2 | OI-04 | BD-6-05 | `canonicalRead` → soak → `canonicalWrite` soak criteria |
| 3 | OI-07 | TD-6-09 | Canonical read assertion-balance consumption contract |
| 4 | OI-10 | TD-6-15 | Phase 6 documentation / task-ID convention |

**No other item is escalated here.** Everything `02A` classified as `EXECUTION GATE ONLY`, `EVIDENCE REQUIRED`, `DEFERRED`, or `CLOSED` keeps that status unchanged.

---

# Decision 1 — OI-03 / BD-6-03

**Decision ID:** `OI-03` (source decision ID `BD-6-03`)

**Decision Title:** EOL / retirement timing for `POST /rates/engine/modify`

### Why This Decision Is Still Open

The **disposition is already decided** — the route must retire. What no rule fixes is *when*. Timing cannot be inferred from the repository, and `02A` records explicitly that a date "cannot be inferred from any of this" and "must not be silently invented."

### Established Evidence

| Fact | Evidence | Quality |
|---|---|---|
| Route exists as `POST engine/modify` → `crs.modifyReservation` | `rates-inventory.controller.ts:108-123` | `PROVEN` |
| It writes the **legacy** `availability` counters | `crs-engine.service.ts:391-402` → `inventory.domain-service.ts:164,181` (`UPDATE availability … reserved = …, available = …`) | `PROVEN` |
| It does **not** call the Availability assertion port | zero occurrences of `assertion` anywhere in `crs-engine.service.ts` | `PROVEN` |
| It therefore mutates reservation stay fields without updating assertion balances — this is the **H-1 dual-truth generator** | `01` H-1 / M-8; `02A` §K.1 re-verified `crs-engine.service.ts:391-402` is the **only** live in-repo legacy counter writer (`reservation.repository.ts:89` injects `InventoryDomainService` but never invokes it) | `PROVEN` |
| It reads `rate_restrictions` as a hard block while the authority reports `UNRESOLVED` | `crs-engine.service.ts:146-196` (hard block), `:244-288` (`blocked` from legacy, `available`/`eligibility` from authority) | `PROVEN` |
| In-repo consumers are fully known; **external** consumers are not | `gateway/ingress/nginx.conf` and `gateway/api-gateway/kong.yml` contain **no consumer registry** and no per-route consumer list | `PROVEN` (absence of registry) / `UNKNOWN` (absence of traffic) |
| Disposition: retire | BR-5-017/018/025; FDS §24.4 (`03:981`) — *the legacy writer ceases when the last CRS booking/modify path is re-pointed* | `CLOSED` |

### What Is Already Decided

- The route **will** be retired (BR-5-017/018/025, FDS §24.4).
- **E-5 is an execution/evidence dependency, not a disposition dependency.** `phase-5/04:261,272` (T5-88): *"inventory unknown ⇒ in-repo retirements proceed, external-facing retirement waits (**disposition unchanged**)."*
- Retirement of this route gates `G-2` (`InventoryDomainService` removal) and the DS-05 external leg.
- Under **no** option may this route become an end state.

### What Is NOT Decided

- **The retirement timing / EOL policy.** — `OWNER TO SPECIFY`

### Available Options

| # | Option | Description | Precondition |
|---|---|---|---|
| **A** | **Retire on E-5 confirmation (no calendar date)** | Route is retired as soon as E-5 returns and shows no external consumer. Timing is defined by an event, not a date. | E-5 returned empty |
| **B** | **Deprecate now, retire on an owner-specified date** | Route stays live with a deprecation signal until the owner's date; retired on that date regardless of E-5 outcome, with residual risk explicitly accepted if E-5 is still open. | `Date: OWNER TO SPECIFY` |
| **C** | **Harden first, then retire** | Interim: the route is changed so stay-field writes go through the assertion port (or the route refuses writes that would not update balances), then retired under A or B. **C is never the end state** — FDS §24.4 requires cessation. | Owner accepts a code change before retirement |

> Note: **C is the only option that requires production code before retirement.** It is listed because `02` established it; selecting it authorises a *plan task*, not an action taken here.

### Consequences

| | A | B | C |
|---|---|---|---|
| H-1 dual-truth generator | stops at retirement | keeps generating until the date | keeps generating until final retirement under A/B |
| External callers | none exist (E-5 empty) → no impact | unaffected until the date | unaffected until the date |
| If E-5 is never returned | retirement is blocked indefinitely | not blocked — date governs | not blocked — date governs |
| Code change required | none | none | **yes** |
| Compliance with FDS §24.4 | immediate | satisfied at the date | satisfied only at final retirement |

### Recommendation

**`02`'s recommendation, preserved:** obtain E-5 first, then **A** if empty; otherwise **B**, with **C** only as an interim if the window is long — never as the end state.

If the owner prefers a decision that does not depend on an evidence item that may never return, **B** is the self-contained choice: it fixes a date and accepts the residual risk explicitly rather than inheriting it implicitly.

### OWNER DECISION REQUIRED

> **Select the EOL/retirement policy for `POST /rates/engine/modify`: `A`, `B`, or `C`.**
> **If `B` (or `C` as interim): `Date: OWNER TO SPECIFY`.**
> **If `C`: confirm the hardening scope authorises a plan task only.**

### Downstream Impact

- Reservations Phase 9: `H-7` divergence between CRS and Front Office disappears only after this retirement (`02` §R).
- Front Office: unchanged — already on the authority port.
- `G-2` `InventoryDomainService` removal becomes executable.
- `TD-6-02` (`crs.modifyReservation` port-vs-retire) inherits its timing from this decision.
- `G-15` retired-route delay allowlist retires with this route.

### What Becomes Unblocked After Decision

- §H cutover exit gate 2 (E-5 answered **or** a BD-6-03 date set) — satisfied.
- §H cutover exit gate 6 (H-1 generator eliminated) — schedulable.
- Plan task for `TD-6-02` / `BD-6-03` execution — scope and order fixed.

---

# Decision 2 — OI-04 / BD-6-05

**Decision ID:** `OI-04` (source decision ID `BD-6-05`)

**Decision Title:** What qualifies as a successful `canonicalRead` → soak → `canonicalWrite`

### Why This Decision Is Still Open

The **sequence** is ratified and preserved verbatim in four places. The **criteria are defined nowhere.** `02A` verified: FDS §28.3 (`03:1105-1110`), §28.5 (`03:1120`), AC-39 (`03:1225`), `phase-5/04:163`, and `.env.example:101-106` all preserve the word "soak" without a duration or threshold. The only *defined* soak in the corpus — **T5-10 "parity soak"** (`04:405-413`) — is a different artifact (flag-ON value parity between matrix and snapshot for the `gba.a3.authoritative` gate) and **must not be reused as an assumption.**

### Established Evidence

| Fact | Evidence | Quality |
|---|---|---|
| Sequence `canonicalRead` → soak → `canonicalWrite` is ratified and must never be altered; never both at once | FDS §28.5 (`03:1120`), AC-39 (`03:1225`), `phase-5/04:163`, `.env.example:105-106` | `PROVEN` |
| `canonicalRead` selects the pickup **read** source; default OFF = legacy read; code comment: *"enable after the parity window"*; the canonical query **aliases every legacy column name so the response shape is byte-identical across both paths** | `allotment.controller.ts:302-313` | `PROVEN` |
| `canonicalWrite` gates the legacy flip; default OFF = **both ledgers written** (BLK-3 cutover exception); ON = canonical write only, legacy read-only | `reservation-pickup-cascade.service.ts:94-102` | `PROVEN` |
| Therefore during the soak window **both ledgers are written in lockstep and directly comparable** | combination of the two above | `PROVEN` |
| A purpose-built detector already exists: `LEGACY_DRIFT` compares legacy `allotment_pickups` vs canonical, **field-by-field** | `gba-reconciliation.service.ts:347-392` | `PROVEN` |
| That detector is currently **exact-match (zero tolerance)** — any unequal field produces a finding | `gba-reconciliation.service.ts:379` (`if (left !== right) mismatches[field] = …`) | `PROVEN` |
| It runs **hourly**, is **report-only** (`automaticRepair` literal `false`), **hotel-scoped**, and gated by `gba.reconciliation.enabled` (currently false) | `gba-reconciliation.service.ts:23,131,162-201`; FDS §28.1 | `PROVEN` |
| Read-path parity is directly measurable because both responses are byte-identical | `allotment.controller.ts:302-312` | `PROVEN` |
| Rollback requires no code: `canonicalRead` OFF restores legacy read (the default); `canonicalWrite` stays OFF so dual-write continues — no data is lost | `allotment.controller.ts:313`; `reservation-pickup-cascade.service.ts:102`; FDS §24.7 | `PROVEN` |
| `G-10` (legacy `allotment_pickups` removal) is gated on `canonicalWrite` ON | `02` §I G-10 | `CLOSED` |

### What Is Already Decided

- Order: `canonicalRead` ON → soak → `canonicalWrite` ON. Never altered; never both at once (FDS §28.3/§28.5, P-21).
- Instrument candidate: the already-built `LEGACY_DRIFT` detector — no new tool is required by the ratified rules.
- Soak exit is a **gate**, not a scope item: `canonicalWrite` may not be turned on until criteria exist and are met.

### What Is NOT Decided

The **values** of five soak parameters. None is defined in any ratified document; **all five are `OWNER TO SPECIFY`.**

### Available Options

The choice set for each parameter is limited to the shapes below. **No numeric values are proposed** — duration, number of days, percentage threshold, mismatch threshold, sample size, and monitoring period are all left for the owner.

| # | Parameter | Options (shape only — **no values proposed**) | Value |
|---|---|---|---|
| 1 | **Minimum soak duration** | calendar floor · event-count floor · both (the greater of) | `OWNER TO SPECIFY` |
| 2 | **Acceptable discrepancy threshold** | exact-match (current detector behavior, zero tolerance) · findings ≤ N · findings ≤ X% of rows compared | `OWNER TO SPECIFY` |
| 3 | **Required evidence** | hourly `LEGACY_DRIFT` reports across the full window · read-path parity between legacy and canonical responses (byte-identical shape makes this measurable) · both | `OWNER TO SPECIFY` |
| 4 | **Failure condition** | any finding · findings above the threshold · detector `ERROR` status · no data compared during the window | `OWNER TO SPECIFY` |
| 5 | **Restart / rollback condition** | a failure resets the clock · `canonicalRead` OFF reverts to legacy read (no code change, FDS §24.7) · `canonicalWrite` is never flipped until criteria pass | `OWNER TO SPECIFY` |

### Consequences

- Choosing an **exact-match / zero-tolerance** threshold (parameter 2) matches what the detector does today and demands a fully clean window; any looser threshold must be explicitly ratified because it changes what `LEGACY_DRIFT` findings mean.
- Choosing **both** read-path parity and `LEGACY_DRIFT` as required evidence uses only what already exists; adding any other evidence requirement implies a new instrument (outside this decision's authority).
- **Every rollback option is operational, not code**: `canonicalRead` OFF restores the legacy read, and `canonicalWrite` remains OFF throughout the soak, so the legacy ledger is never frozen and `G-10` removal stays gated.

### Recommendation *(separate from the OWNER DECISION — not a choice made here)*

**Shape** — as established by `02` and preserved here: a **parity-threshold window on `LEGACY_DRIFT`, with a minimum calendar floor**, using the already-built reconciler as the soak instrument rather than building a new one; `gba.reconciliation.enabled` becomes a **precondition** of the soak (a sequencing observation, not a new rule).

> **The recommendation deliberately contains no numbers.** Duration, threshold, sample size, and monitoring period remain `OWNER TO SPECIFY`.

### OWNER DECISION REQUIRED

> **Specify parameters 1–5:**
> **1. Minimum soak duration: `OWNER TO SPECIFY`**
> **2. Acceptable discrepancy threshold: `OWNER TO SPECIFY`**
> **3. Required evidence: `OWNER TO SPECIFY`**
> **4. Failure condition: `OWNER TO SPECIFY`**
> **5. Restart / rollback condition: `OWNER TO SPECIFY`**
>
> **Accept or replace the recommendation above.**

### Downstream Impact

- `TD-6-11` / `OI-28` — activation of `gba.reconciliation.enabled` becomes a **precondition** of the soak (currently `DEFERRED` with no ratified order).
- `G-10` — legacy `allotment_pickups` removal remains blocked until `canonicalWrite` ON, which this decision gates.
- `BD-6-04` — `gba.consumers.cascade` MUST be ON at deploy (P-21) regardless of this decision; cascade ON begins writing the canonical ledger, so this soak should be sequenced with it.
- §H cutover exit gate 8 — "soak criteria written" is satisfied.

### What Becomes Unblocked After Decision

- Scheduling of the `canonicalRead` → soak → `canonicalWrite` sequence as plan tasks (scope already fixed; only gate values were missing).
- Activation path for `gba.reconciliation.enabled`.
- Any forward movement toward `G-10` legacy-ledger removal.

---

# Decision 3 — OI-07 / TD-6-09

**Decision ID:** `OI-07` (source decision ID `TD-6-09`, finding `D-AUD-01`)

**Decision Title:** Should the canonical Availability read contract materialise `availability_assertion_balances` for `ASSERTION_MANAGED` reservations?

### Why This Decision Is Still Open

`02A` verified both readings are defensible and that **nothing in the corpus declares which is intended**: P-11/P-12 + FDS §26.2 argue for inclusion, while the ratified three-term formula (which predates assertion balances) plus the existence of a *separate* sanctioned balance read (`gba.pickup.twoLayerConsult`) argue against. It is a change to a Phase 5–published read contract that is operator-visible — therefore an owner sign-off, not a technical default.

### Established Evidence

| Fact | Evidence | Quality |
|---|---|---|
| Canonical read combination has **three** terms: `consumption = reservationConsumption + gbaRemaining + allotmentRemaining` | `snapshot-calculator.ts:14`; `phase-4/02_CURRENT_ARCHITECTURE.md:131`; `phase-4/13_FINAL_DOMAIN_SPECIFICATION.md:448` | `PROVEN` |
| **No reader of `availability_assertion_balances` exists anywhere in the read path** | repo-wide; only readers are in `availability-assertion.service.ts` (write path) | `PROVEN` |
| The **write gate is correct**: assert-time `remaining = day.sellableAvailable − balances.get(stayDate)` | `availability-assertion.service.ts:894-900` | `PROVEN` |
| First-touch assigns population on assert, so essentially every reservation created since the authority went live is `ASSERTION_MANAGED` | `availability-assertion.service.ts:271-275` | `PROVEN` |
| The omission is **operator-visible**: matrix authority branch sets `available: fact.sellableAvailable`; the page renders `cell.available` | `availability-sales.controller.ts:474,494,531,539`; `AvailabilityPage.tsx:27,788,1053`; flag currently ON (`.env:68`) | `PROVEN` |
| **G5 no-dual-count is satisfied** — each Reservation is counted by exactly one of {legacy source, assertion balances}; excluding `ASSERTION_MANAGED` rows from the source is therefore *right* | `availability-phase3-implementation-plan.md:613,829`; `availability-phase3-readiness-review.md:194` | `PROVEN` |
| Assertion balances have their **own** sanctioned read (`gba.pickup.twoLayerConsult`, default OFF) | FDS `:691`; `.env.example:104` | `PROVEN` |
| Ratified rules: exactly four fact kinds, fourth = "reservation commitment" (P-11); Availability is the sole source of the sellable number, *"no independent 'available' number anywhere"* (P-12); balances are **domain state**, "the current business truth" (FDS §26.2) | `03:105,389,394`; `03:106`; `03:1032` | `CLOSED` |
| Reconciliation compares **like-for-like**: legacy `available` also omits assertion consumption because the only live legacy counter writer is the external modify path | `02A` §K.1 — `crs-engine.service.ts:391-402` (M-8); `reservation.repository.ts:89` injects but never invokes; `reserveRooms` has no callers | `PROVEN` |
| T5-10 parity soak compares matrix against snapshot only, seeds no `ASSERTION_MANAGED` rows — it cannot detect this | `t510-parity-soak.spec.ts` | `PROVEN` |

### What Is Already Decided

- **Option "declare a post-assertion contract requiring every reader to subtract balances" is RULED OUT** by ratified rules — it creates a second combination engine, violates P-11/P-12, and is impossible for frontends. **It is not offered.** (Preserved from `02`/`02A`.)
- Excluding `ASSERTION_MANAGED` rows from `reservationConsumption` is correct (G5).
- The write side is correct and is not part of this decision.
- G5/no-dual-count is not at stake under either remaining option.

### What Is NOT Decided

Whether the **read** materialises assertion balances.

### Available Options

| # | Option | Description |
|---|---|---|
| **A** | **Canonical read consumes assertion balances** | Add a provenance-labelled additive field (e.g. `assertionBalanceConsumption`) read from `availability_assertion_balances` and combined by the snapshot calculator into `consumption` / `sellableAvailable`. The existing `reservationConsumption` field stays **byte-identical** so no current consumer breaks. Requires a dated `annotate` note against FDS §11.3 (annotate, never rewrite). |
| **B** | **Canonical read remains projection-based** | Read stays three-term: `sellableAvailable` = capacity − *legacy-population* consumption − GBA − allotment. The gap is documented as a known characteristic of the contract. No field changes. |

### Consequences

| Surface | **A** — read consumes balances | **B** — read stays projection-based |
|---|---|---|
| `snapshot.sellableAvailable` | becomes `capacity − N` for N `ASSERTION_MANAGED` reservations (today it equals `capacity`) — matches assert-time `remaining` | unchanged; over-reports by N for `ASSERTION_MANAGED` populations |
| `GET /availability/matrix` (`available` cell) | moves to the true value in lockstep with the snapshot — existing matrix↔snapshot parity relation is preserved | unchanged; keeps reporting an inflated figure while the flag is ON |
| `AvailabilityPage` (operator view) | shows the genuinely sellable count | continues to show a count that includes rooms already committed |
| `/reconciliation` | `expected` moves; **any baseline captured before the change must be re-captured** | `MATCH` remains like-for-like (valid for **projection drift** per REQ-25.1) but does **not** mean "correctly sellable" |
| Assertion-integrity evidence | unaffected — **I-2 is a separate comparator** and is still absent under either option | unaffected — same |
| Cutover | §H exit gate 3 (TD-6-09 confirmed and implemented with DB-backed proof) becomes satisfiable; capacity-grade baselines become available | capacity-grade baselines **cannot** be produced from this comparator; publishing authority numbers would publish an inflated `sellableAvailable` |
| Verification | one **DB-backed** test per surface (snapshot, matrix, reconciliation) seeding `ASSERTION_MANAGED` balances; existing mocked specs catch none of it | a dated documented-gap note |
| Contract change | yes — FDS §11.3 amendment note (annotation) | none |

### Recommendation *(separate from the OWNER DECISION — not a choice made here)*

**Option A**, as an **additive** field, with the legacy-population field left byte-identical so no existing consumer breaks — because it makes the read agree with the already-correct write gate and with P-11/P-12, and because `02` showed option C-equivalent (defer) leaves §H exit gate 3 unsatisfiable.

> Because this amends a ratified, published read contract, it must not be slipped in: **engineering owner signs the change; product owner acknowledges it** (`sellableAvailable` is operator-visible).

### OWNER DECISION REQUIRED

> **Approve the canonical read contract:**
> **`A` — canonical read consumes `availability_assertion_balances` (additive field + FDS §11.3 annotation), or**
> **`B` — canonical read remains projection-based (documented gap).**
>
> **Note: this is also a downstream-notice decision — Reservations Phase 9 consumes `sellableAvailable`.**

### Downstream Impact

- **Reservations rebuild (Phase 9):** consumes `sellableAvailable`; must be notified either way (`02` §R).
- **Testing policy:** if A, DB-backed tests become mandatory for snapshot, matrix, and reconciliation (`02` §R — existing mocked specs catch neither).
- **Reconciliation severities:** if A, `/reconciliation` variances shift upward; any pre-existing captured baseline is invalidated and must be re-captured.
- **Flags:** no flag change is implied by either option (G-5 stands).

### What Becomes Unblocked After Decision

- §H cutover exit gate 3 — "TD-6-09 confirmed and, if approved, implemented with DB-backed proof".
- Capture of **capacity-grade** cutover baselines (`02A` §I, refined two-tier rule).
- `02A` §K.1's refined sequencing rule becomes fully executable.
- Plan task for the read-contract change (if A) or the documented-gap annotation (if B).

---

# Decision 4 — OI-10 / TD-6-15

**Decision ID:** `OI-10` (source decision ID `TD-6-15`, blocker `X-7`)

**Decision Title:** Phase 6 documentation and task-ID convention

### Why This Decision Is Still Open

`01` §C.4 D-4/D-5 established that **no rule in either precedent yields a unique Phase 6 list**, and D-6 forbids inheriting `T5-nn` / `BLK-P5-nn` / `E-n` / `S3R-nnn`. `02` deliberately did not decide it (the brief prohibited inventing document trees). `02A` classified it as the single item needed **before a plan document can be authored**, while confirming it does not block the definition of scope, dependency, behavior, verification, or order — those are already enumerated in `02` §R.

This is an **authoring/process** decision, not a business rule.

### Established Evidence

| Fact | Evidence | Quality |
|---|---|---|
| Phase 6 documents that exist today | `01_FORENSIC_AUDIT.md`, `02_DECISION_RESOLUTION_ANALYSIS.md`, `02A_OPEN_DECISION_EVIDENCE_RESOLUTION.md`, `02B_OWNER_DECISION_RESOLUTION.md` (this file) | `PROVEN` |
| Inheriting Phase 5 ID namespaces is forbidden | `01` §C.4 D-6 | `CLOSED` |
| Phase 5 used `T5-nn` (90 tasks), `BLK-P5-nn`, `E-1…E-8`, `S3R-nnn` | `phase-5/04_IMPLEMENTATION_PLAN.md` §30–§33 | `PROVEN` |
| Phase 6 decision IDs already exist and are stable: `BD-6-01…10`, `TD-6-01…15`, `CO-6-01…07`, plus `OI-01…OI-30` (02A-local, explicitly carrying **no** phase/task authority) | `02` §C/§D/§H; `02A` §C | `PROVEN` |
| `02` §R: plan scope is enumerated; **"Nothing else may enter the plan without a decision ID."** | `02` §R | `CLOSED` |
| Phase 6's exit requires **recorded** evidence: E-8 reconciled baselines; §H exit gates 1–9 are all "recorded evidence"; census output (TD-6-04) | FDS §31 E-8; `02` §H; `02` CO-6-02 | `PROVEN` |
| Next free document number in the Phase 6 series is `03` | existing filenames | `PROVEN` |

### What Is Already Decided

- Next document (when authorised) is numbered `03`.
- Decision IDs `BD-6-*` / `TD-6-*` / `CO-6-*` are **not renumbered** — they are the traceability spine referenced by `02` §R and `02A` §K.
- `OI-*` identifiers remain `02A`-local and are **not** a task namespace.
- No artificial "stages" and no workflow beyond what the ratified sequencing already implies (DS-01 → DS-03 → DS-05; `canonicalRead` → soak → `canonicalWrite`).

### What Is NOT Decided

- The document set, the task-ID prefix, and whether gate/evidence namespaces are introduced.

### Available Options

| # | Option | Documents | Task IDs | Gate / evidence IDs | Traceability |
|---|---|---|---|---|---|
| **A** | **Minimum — one document** | `03_IMPLEMENTATION_PLAN.md` only | `T6-01 … T6-nn` | **None introduced.** Gates cited by their existing ratified IDs (`CO-6-*`, `BR-5-0nn`, FDS §24.2 / §28.3, `02A` §H exit gates 1–9); evidence cited inline in the task record. | each task carries the `BD-6-*` / `TD-6-*` / `CO-6-*` ID it implements |
| **B** | **Plan + evidence register — two documents** | `03_IMPLEMENTATION_PLAN.md` + `04_EXECUTION_EVIDENCE.md` | `T6-01 … T6-nn` | `E6-01 … E6-nn` for recorded runs; gates still cited by existing ratified IDs (no new `BLK-P6-nn` namespace) | task ↔ evidence entry ↔ decision ID |
| **C** | **Phase 5 mirror — three documents** | `03_IMPLEMENTATION_PLAN.md`, `04_EXECUTION_EVIDENCE.md`, `05_READINESS_REVIEW.md` | `T6-01 … T6-nn` | `E6-nn` + a readiness review | task ↔ evidence ↔ review ↔ decision ID |

All three: avoid collision with `T5-nn` / `BLK-P5-nn` / `E-n` / `S3R-nnn`; are stable across implementation; introduce no stage model.

### Consequences

| | A | B | C |
|---|---|---|---|
| Document count | 1 | 2 | 3 |
| Where E-8 / §H "recorded evidence" lives | inside the plan's task records | dedicated register | dedicated register |
| Risk of drift between plan and evidence | higher (single document grows) | low | low, but adds a third artifact before execution exists |
| New namespaces invented | none | one (`E6-nn`) | two (`E6-nn` + readiness) |
| Implies undocumented workflow | no | no | **partially** — a readiness review implies a review stage that no ratified Phase 6 rule describes |

### Recommendation *(separate from the OWNER DECISION — not a choice made here)*

**Option B.** Phase 6's exit is defined by *recorded* evidence (E-8 reconciled baselines; `02` §H gates 1–9; census output), so a place to record it is required by the ratified rules rather than invented here. It adds one document and one identifier namespace, introduces no stage, and implies no workflow.

**Option A** is correct if the owner prefers the smallest possible surface and is willing to record evidence inline in task records.

**Option C** is listed for completeness but is **not recommended** — the readiness review would imply a workflow stage that no Phase 6 rule documents.

### OWNER DECISION REQUIRED

> **Approve the Phase 6 convention: `A`, `B`, or `C`.**
> **Task-ID prefix: `T6-nn` (proposed). OWNER MAY SPECIFY OTHERWISE.**
> **Evidence-ID prefix: none (A) / `E6-nn` (B, C).**

### Downstream Impact

- Determines the structure of `03_IMPLEMENTATION_PLAN.md`, which is **not** created by this document.
- Fixes the identifier collision boundary against Phase 5 (`T5-nn`, `BLK-P5-nn`, `E-n`, `S3R-nnn`).
- Preserves traceability: every future plan task must cite the decision ID it implements (`02` §R).

### What Becomes Unblocked After Decision

- **Authoring of `03_IMPLEMENTATION_PLAN.md`** — the sole precondition identified by `02A` §L.3 condition C1.
- Nothing else: all other `02A` gates are carried *inside* the plan.

---

# Summary of what is NOT being decided here

Preserved unchanged from `02A` — **not escalated, not converted**:

| Status in `02A` | Items |
|---|---|
| `CLOSED` (7) | OI-01, OI-08, OI-09, OI-12, OI-14, OI-17, OI-19 |
| `EVIDENCE REQUIRED` (8) | OI-02, OI-16, OI-18, OI-20, OI-21, OI-22, OI-23, OI-24 |
| `EXECUTION GATE ONLY` (3) | OI-13, OI-15, OI-25 |
| `DEFERRED` (7) | OI-05, OI-06, OI-11, OI-27, OI-28, OI-29, OI-30 |
| `NOT APPLICABLE` (1) | OI-26 |
| `PLANNING BLOCKER` (0) | — |

No execution gate has been turned into an owner decision. No evidence gap has been turned into an owner decision. No new decision has been invented.

---

## OWNER RESPONSE REQUIRED

**OI-03:**
Decision: ______________________________
If B (or C interim): Date: `OWNER TO SPECIFY` ______________________________

**OI-04:**
Decision:
1. Minimum soak duration: ______________________________
2. Acceptable discrepancy threshold: ______________________________
3. Required evidence: ______________________________
4. Failure condition: ______________________________
5. Restart / rollback condition: ______________________________

**OI-07:**
Decision:  ☐ **A** — canonical read consumes assertion balances (additive field + FDS §11.3 annotation)
☐ **B** — canonical read remains projection-based (documented gap)

**OI-10:**
Decision:  ☐ **A** — plan only (`03_IMPLEMENTATION_PLAN.md`, `T6-nn`, no evidence namespace)
☐ **B** — plan + evidence (`+04_EXECUTION_EVIDENCE.md`, `E6-nn`) *(recommended)*
☐ **C** — plan + evidence + readiness review
Task-ID prefix: `T6-nn` *(proposed)* / other: ______________________________

---

## Final Status

- **Four owner decisions prepared** — OI-03, OI-04, OI-07, OI-10. Nothing else was escalated.
- **No implementation performed.**
- **No code, schema, test, or flag changes.** No migration, no DB write, no reconciliation run, no cutover, no legacy deletion, no legacy writer retirement.
- **No Implementation Plan and no implementation tasks created.** `03_IMPLEMENTATION_PLAN.md` was not created.
- **No additional decisions invented.** Every other `02A` item retains its existing status.
- **No prior document modified** — `01`, `02`, and `02A` are untouched.

**Next action occurs only after the owner explicitly resolves the four decisions. This document does not proceed to Implementation Planning.**

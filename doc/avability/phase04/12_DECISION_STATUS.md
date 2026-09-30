# Phase 4 — Decision Status Register

**Phase:** 4 — GBA / Allotment Integration (Decisions only — NOT implementation)
**Date:** 2026-09-30
**Companion documents:** `10_DECISION_RESOLUTION.md` (full analysis, options, evidence) · `11_TARGET_BUSINESS_RULES.md` (consolidated target rules)
**Scope honored:** 0 code / schema / migration / backfill / API changes; no decisions silently made — every non-evidence-forced choice appears in §2.

---

## 1. Decision Status Matrix

21 decisions (D-1…D-15 + S-1…S-6) + 1 sub-decision (D-6a) = 22 items.

| ID | Decision (short) | Status | Selected / Recommended rule | Depends on | Blocks implementation? |
|---|---|---|---|---|---|
| **D-1** | Authoritative GBA data world | **DECIDED** | New world authoritative; legacy `RETAIN` read-only → Phase-11 removal | — | Blocks Phase 11 legacy removal + migration framing |
| **D-6** | Authoritative availability number | **DECIDED** | Availability (A1) is sole sellable-capacity authority; no independent computations (A3/F-18 must derive) | — | **YES — gates Phase 6 integration & D-7** |
| **D-6a** | *Sub:* A3 transition path | **RECOMMENDED — USER CONFIRMATION** | Reimplement A3 on Availability (A); labeled interim view only (B) | D-6 | Gates A3/Activities rework |
| **D-8** | Concurrency guarantee | **DECIDED** (invariants) | No lost updates; conflict → error; picked ≤ contracted; exactly-once; version monotonicity. *Mechanism (optimistic vs pessimistic): `DEFERRED` to implementation plan* | — | **YES — gates all counter/pickup work** |
| **D-5** | Event consumption | **DECIDED** | No published event without a consumer; availability invalidation + GBA reservation consumer mandatory | — | **YES — gates D-4, real-time invalidation** |
| **D-2** | Schema reproducibility | **RECOMMENDED — USER CONFIRMATION** | Domain requirement DECIDED (declare everything executed); mechanism = forward migration after live-DB verification (A) | **user action: live DB check** | **YES — top gate (schema-dependent work: D-15, D-3, D-13, D-4, migrations)** |
| **D-15** | Block lifecycle reachability | **DECIDED** | Explicit T-6 transitions only; `DEFINITE`/`OPEN_FOR_PICKUP` hold inventory; pickup requires eligible state; options C/D rejected | D-2, D-6 | **YES — blocks A1 block-consumption correctness** |
| **D-7** | Pickup ↔ availability/assertion | **DECIDED** | Two-layer check (pool capacity + full reservation assertion); B/C rejected. *Mechanism: `DEFERRED`* | D-6, D-15 | **YES — closes A5/L-14 bypass** |
| **D-9** | Pickup transaction boundary | **DECIDED** | Pickup = one atomic unit (reservation + record + counters); partial commit prohibited | D-8 | **YES — gates pickup rewrite** |
| **D-10** | Fabricated voucher reservation id | Rule **DECIDED** / mechanism **RECOMMENDED — USER CONFIRMATION** | No synthetic ids (decided); consume creates/links real reservation (B, recommended) | — | **YES — gates S-1, voucher reporting** |
| **S-4** | Stop-sale enforcement uniformity | **DECIDED** | Stop sale blocks ALL allotment intake paths; allotment-only scope; quantity untouched; one guard formula | D-6 | **YES — gates S-1** |
| **S-1** | Canonical allotment pickup mechanism | **RECOMMENDED — USER CONFIRMATION** | One canonical pickup record; voucher = authorization artifact (A) | D-10, S-4, D-9 | **YES — gates pickup ledger unification** |
| **S-5** | Used-voucher cancel quota policy | **USER DECISION REQUIRED** | Follow reservation lifecycle (A, recommended) | S-1, D-10, D-4 | Gates used-voucher cancel behavior |
| **D-3** | Cut-off wash / rolling release | **USER DECISION REQUIRED** | Implement scheduled wash per T-1/T-2, date granularity (A, recommended) | D-2, D-15, D-5 | Gates wash/automation scope; until decided, cutoff/release fields = non-authoritative |
| **D-4** | FO checkout ↔ pickup status (L-13) | **RECOMMENDED — USER CONFIRMATION** | Derived pickup state via GBA reservation-event consumer (B structure + C semantics); FO direct writes removed | D-5, D-2 | **YES — closes L-13** |
| **D-11** | Guest resolution by name | **USER DECISION REQUIRED** | Match by explicit id/email, else create; never `full_name ILIKE` (C, recommended) | — | Gates guest-facing pickup changes |
| **D-12** | Block-release HTTP verb | **DECIDED** | `POST` canonical; frontend corrected; no `PUT` alias | — | Blocks only the 405 fix (trivial) |
| **D-13** | Shoulder schema + semantics | Base **DECIDED** / differentiators **USER DECISION REQUIRED** | Base: ordinary allocations in extended range, declared qty, guarded, evented. Differentiators: attrition base / wash interaction / pickup access (uniform recommended) | D-2, D-9 | **YES — schema rides on D-2; feature fixes gated on differentiators** |
| **D-14** | Attrition activate vs remove | **USER DECISION REQUIRED** | Retain + implement per T-7 (A, recommended) | D-3 (wash-time assessment) | Gates attrition scope (feature vs deletion) |
| **S-2** | Allotment overbooking | **USER DECISION REQUIRED** | No overbooking / hard quota (A, recommended) | D-6 | Minor — clarifies quota cap rule |
| **S-3** | Contract-type vocabulary | **RECOMMENDED — USER CONFIRMATION** | Spec vocabulary canonical + published mapping from stored values (A) | D-1 | Gates rule vocabulary for D-3/D-6/S-4 docs & later renames |
| **S-6** | Tenant-scoped statements + header override (F-8) | **DECIDED** | Every statement hotel-scoped; privileged-only override; dev bypass never on real data | — | **YES — security invariant for all write work** |

---

## 2. USER DECISIONS REQUIRED

### 2a. Business policy decisions (6) — implementation cannot proceed correctly without these

| # | Decision | Options (consequence summary) | Recommendation |
|---|---|---|---|
| **1** | **D-3 — Wash/rolling release: build, manual-only, or defer?** | **A** implement scheduled wash (restores T-1/T-2; inventory reclaims automatically; needs scheduler work) · **B** manual-only + delete/relabel dead release fields (honest but diverges from spec; inventory held until operator acts) · **C** defer (keeps the data trap: users enter cutoff dates that do nothing) | **A**, with **date-granularity** timing. Evidence: fields stored but never evaluated (`release-window.value-object.ts:26`, `05_…:125-126`); spec `:1146` GUARANTEED-no-wash invariant. If C, record as explicit deferral with the trap acknowledged |
| **2** | **D-11 — Guest resolution** | **A** keep name-match (distinct same-name guests silently merge) · **B** always create (duplicate-guest explosion) · **C** explicit id → hotel-scoped email → create new | **C**. Evidence: `prisma-reservation-association.adapter.ts:83-97,245-260`; spec silent (B-13) |
| **3** | **D-13 — Shoulder-day differentiators** | Policy set: (a) shoulder rooms in attrition base? (b) wash/cutoff applies as core days? (c) pickups on shoulder dates unrestricted? | **Uniform: yes/yes/yes** (shoulder = ordinary days). Alternatives only if shoulder days are intended as non-sellable buffer — then say so explicitly. Base semantics already decided (ordinary allocations) |
| **4** | **D-14 — Attrition** | **A** implement per T-7 (threshold 80%, shortfall, master-folio penalty) · **B** remove service + column (honest schema, loses revenue protection) | **A** — spec treats it as a core guarantee (`05_…:113`); B only if attrition-bearing contracts are not sold |
| **5** | **S-2 — Allotment overbooking** | **A** hard quota ≤ physical (current behavior, made explicit) · **B** overbooking with tolerance policy (needs explicit flagged fact in Availability) · **C** leave unspecified | **A** — matches existing guards (`allotment.aggregate.ts:424-426`); keeps D-6's single authority clean. Spec open question `gba-domain-spec.md:1720` |
| **6** | **S-5 — Cancel a USED voucher: restore quota?** | **A** restore iff linked reservation cancelled (follows reservation truth) · **B** never restore (current; permanently burns quota) · **C** always restore (can over-sell under a live reservation) | **A** — only option without a deterministic failure mode. Evidence: undecided stub `allotment.aggregate.ts:298-303` |

### 2b. Confirmations required (6) — evidence narrows to one option, but the choice is yours

| # | Decision | What needs confirming | Consequence if declined |
|---|---|---|---|
| **1** | **D-2 — Schema mechanism** | Confirm **A: forward migration after live-DB verification** (vs accepting `db push` for GBA, or removing shadow artifacts). **Prerequisite action first:** read-only `information_schema` check — do the GBA tables, `allotment_pickups`, `group_blocks.shoulder_days_*` exist in the target environment? | Unreproducible environments (B) or premature behavior change (C). **This is the top gate** (`09_…` §1) |
| **2** | **D-6a — A3 migration path** | Confirm **A: rebuild Activities matrix on Availability** (vs B: labeled interim view) | A3 keeps diverging until Phase 6 |
| **3** | **D-4 — L-13 close-out** | Confirm retiring the Phase-3 "KEEP" verdict: **pickup state derived via GBA event consumer; FO direct GBA writes removed** | L-13 stays open; silent-failure status writes persist |
| **4** | **D-10 — Voucher consume mechanism** | Confirm **B: consume creates/links a real reservation** (vs A: require caller-supplied id — verify frontend callers first, read-only) | Fabrication rule stands either way; B decides *who* creates the reservation |
| **5** | **S-1 — Pickup mechanism unification** | Confirm **A: one canonical pickup record, voucher as authorization** (vs B: voucher-only, deprecating direct pickup — requires that manual intake is not a real need) | Two ledgers persist; every rule written twice |
| **6** | **S-3 — Contract-type vocabulary** | Confirm **A: spec vocabulary canonical** (`ROLLING_RELEASE`/`GUARANTEED_BLOCK`/`FREE_SALE` + mapping) vs **B: code vocabulary canonical** (spec amended) | Wash-by-type (D-3) and availability filters (D-6) stay ambiguous across two vocabularies |

---

## 3. Counts

| Status | Count | IDs |
|---|---|---|
| `DECIDED` | **10** | D-1, D-5, D-6, D-7, D-8 (invariants), D-9, D-12, D-15, S-4, S-6 |
| `RECOMMENDED — USER CONFIRMATION REQUIRED` | **6** | D-2, D-4, D-6a, D-10 (mechanism), S-1, S-3 |
| `USER DECISION REQUIRED` | **6** | D-3, D-11, D-13 (differentiators), D-14, S-2, S-5 |
| `DEFERRED` (sub-items only) | **2** | D-8 mechanism, D-7 mechanism (+ D-15 command surface) — implementation-plan scope |
| **Total decision items** | **22** | 15 original (D-1…D-15) + 6 supplementary (S-1…S-6) + 1 sub-decision (D-6a) |
| Target rules issued (`11_…`) | **94** | 78 DECIDED · 3 RECOMMENDED · 11 PENDING USER · 2 DEFERRED |
| Decisions skipped | **0** | all D-1…D-15 addressed |

---

## 4. Gate Status After Decisions

```
╔═══════════════════════════════════════════════════════════════════════╗
║  PHASE 4 DECISION RESOLUTION:  COMPLETE                               ║
║  PHASE 4 IMPLEMENTATION READINESS:  BLOCKED — 6 USER DECISIONS +      ║
║  6 CONFIRMATIONS + LIVE-DB VERIFICATION PREREQUISITE                  ║
╚═══════════════════════════════════════════════════════════════════════╝
```

| Gate criterion | Status |
|---|---|
| D-1…D-15 all resolved (no skips) | **PASS** — `10_…` §2 |
| Dependencies identified, resolved in order | **PASS** — `10_…` §1 |
| Non-evidence choices marked, not silently taken | **PASS** — §2 above (6 + 6) |
| Target rules consolidated, 15 sections, target-only | **PASS** — `11_…` |
| Concurrency expressed as business invariants only | **PASS** — `11_…` §11; no lock syntax |
| Hotel isolation rule cross-cutting | **PASS** — `11_…` §14 + `10_…` S-6 |
| No code/schema/migration/backfill changes | **PASS** — 3 documents written; read-only inspection only |
| Implementation may start | **BLOCKED** — see below |

**What blocks implementation start (priority order):**
1. **User action:** live-DB read-only verification (unblocks D-2 mechanism → everything schema-shaped).
2. **D-2 confirmation** (top gate per `09_…` §1).
3. **D-8 + D-9 already DECIDED** — the pickup correctness contract is now available to implement.
4. **D-6 confirmed** (authority + D-6a path) — Phase 6 consumers need it.
5. Remaining confirmations (D-4, D-10, S-1, S-3) and user decisions (D-3, D-11, D-13, D-14, S-2, S-5) gate their respective feature scopes but not the overall start — except **S-6, D-15, D-7** which are decided and may be implemented once D-2 clears.

---

## 5. Routed Out of This Register (not decisions — tracked, not skipped)

| Item | Routed to |
|---|---|
| F-10 — 0 tests for 25 commands | Implementation phase pre-work (test plan) |
| F-12 — `nextBlockCode` bug | Code fix in implementation phase |
| D-8/D-7/D-15 mechanism & command-surface details | Implementation plan (`DEFERRED` sub-items) |
| B-9 column removal (`reservations.group_block_id` etc.) | Phase 11 migration decision (rule: pickup records authoritative — `11_…` TR-9.2) |
| Spec open questions `gba-domain-spec.md:1716-1728` (master-folio ownership, BEO, analytics real-time, aggregate limits, grid storage, voucher modification, CRS port shape) | Backlog — none block D-1…D-15 |
| F-8 header-override hardening mechanics | Implementation of decided S-6 rule |

---

**End of `12_DECISION_STATUS.md`.** Deliverables for this phase: `10_DECISION_RESOLUTION.md`, `11_TARGET_BUSINESS_RULES.md`, `12_DECISION_STATUS.md` — alongside audit docs `01`…`09` in `docs/availability/phase-4/`.

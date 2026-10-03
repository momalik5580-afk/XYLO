# XYLO Availability Phase 5 — Business Rules / Decisions (Stage 2)

**Artifact:** `docs/availability/phase-5/02_BUSINESS_RULES_DECISIONS.md` (artifact **02** of Phase 5)
**Preceded by:** `docs/availability/phase-5/01_FORENSIC_AUDIT.md` (artifact **01**, Stage 1 — read-only forensic audit)
**Status of this document:** decisions only. No source, schema, migration, frontend, test, dependency, or config change is made or implied by the writing of this file.
**Findings input:** F-01…F-27 (audit §10), decision items DS-01…DS-12 (audit §12).

---

## 1. Purpose and Scope

This document converts the Stage 1 forensic findings into **binding business/domain decisions** for Phase 5 (Other Consumers + Frontend Migration). It answers, for each of DS-01…DS-12:

1. what the ratified Phase 1–4 corpus already decides (and is therefore **preserved**, not re-opened),
2. what is decided here as a **new or amended** decision with evidence,
3. what remains **unresolved because evidence is missing** (declared, never papered over), and
4. which **testable business rules**, blocker classes, and **Stage 3 requirements** follow.

In scope: business truth, dispositions of competing sources/consumers, read/write contract choices, eligibility-surfacing duties, scope questions (DS-07/DS-08), test/flag governance, and the blocker/dependency structure for the remainder of Phase 5.

Out of scope (explicitly): implementation of any of the above, code fixes, migrations, test execution, route changes, flag changes, and any restriction *policy* not already ratified or evidenced (see §5 and §30).

---

## 2. How to Read This Document

### 2.1 Decision statuses (used verbatim)

| Status | Meaning |
|---|---|
| `RESOLVED` | Decided in this document; rule(s) issued; no further business decision needed. |
| `PRESERVED FROM PRIOR PHASE` | Already decided by a ratified Phase 1–4 artifact; re-stated with citation; **not re-opened**. |
| `NEW DECISION` | Not previously decided anywhere in the ratified corpus; decided here under the §5 protocol with evidence. |
| `AMENDED DECISION` | A prior decision exists and is explicitly amended here; the superseded text is named (§5 step 3). |
| `UNRESOLVED` | Cannot be decided from available evidence. Named evidence requirements recorded (§30). Never silently defaulted. |
| `BLOCKED` | Cannot be *operationalized* until one or more named dependencies close, even where the rule itself is decided. |

Composite items (DS-01, DS-11, DS-12) carry a **primary status** plus a sub-item table; a sub-item may hold a different status. Blocker classes (`HARD` / `CONDITIONAL` / `NON-BLOCKING` / `DEFERRED`) are registered in §21.

### 2.2 Option matrix (mandatory for every non-trivial decision)

Seven columns, no numeric scores, at least three options including the status quo:

| # | Column | Content |
|---|---|---|
| 1 | **Option** | Name. |
| 2 | **Description** | What would actually happen. |
| 3 | **Evidence / authority alignment** | Ratified rules and file:line evidence touched (supporting or contradicting). |
| 4 | **Business impact** | Effect on truth, eligibility, sellable numbers, guest-facing behavior. |
| 5 | **Operational impact** | Effect on staff workflows, rollout, and day-to-day operation. |
| 6 | **Risk / reversibility** | Failure modes and how easily the choice can be undone. |
| 7 | **Verdict** | Selected / Rejected / Deferred — with the reason. |

### 2.3 Testable business rule format

Every rule issued in §20 (and referenced from §8–§19) carries these labeled elements where applicable:

`ID` · `Source/authority` · `Input` · `Condition` · `Source of truth` · `Calculation` · `Output` · `Error behavior` · `Consumer duty` · `Prohibited behavior`.

### 2.4 Traceability requirement

Every non-trivial decision must close as: **finding → decision item → business rule → affected consumer → Stage 3 requirement** (§23). Every finding F-01…F-27 must appear in §23 and §24; none may disappear.

---

## 3. Authority Hierarchy and Sources Consulted

Decisions below apply this hierarchy (highest first). A lower tier may never override a higher tier, and **legacy behavior is never authoritative merely because it exists**:

1. **User constraint (workspace):** no schema changes for this phase family (AGENTS.md roadmap; Phase 4 deviation C `17_PHASE4_READINESS_REVIEW.md:70`); remote/GitHub is not an authority.
2. **Ratified Phase 1–2 contract** — `docs/enterprise/availability-phase1-2-contract-extract.md` (§4.1–4.5 snapshot contract, §5.1–5.7 assertion contract).
3. **Ratified Phase 3 rulings** — `docs/enterprise/availability-phase3-domain-specification.md`, `availability-phase3-business-rules-decision-sheet.md`, `availability-phase3-stage-b-ratification.md`.
4. **Ratified Phase 4 corpus** — `11_TARGET_BUSINESS_RULES.md` (TR-*), `13_FINAL_DOMAIN_SPECIFICATION.md` (INV-*, §11/§18/§19), `10_DECISION_RESOLUTION.md` (D-*/S-*), `14_IMPLEMENTATION_PLAN.md` (T-*, §8/§11/§13), `15_READINESS_REVIEW.md`, `16_RR_DECISIONS.md`, `17_PHASE4_READINESS_REVIEW.md` (deviations, flags, baselines).
5. **Existing implementation evidence** (file:line) — used to establish what is true today, and to derive absence/conflict handling where two independent implementations agree.
6. **Legacy/product behavior** — lowest; may inform an option but may not by itself establish a rule that contradicts tiers 1–4.

Stage 1 evidence (`01_FORENSIC_AUDIT.md` §15) is cited throughout and was re-verified against source for every critical claim used in a decision below.

---

## 4. Prior-Phase Decisions Preserved (register)

These are **not re-opened** in Phase 5. Each is cited so that Stage 3+ cannot silently change it:

| ID | Preserved decision | Citation |
|---|---|---|
| P-1 | Snapshot is read-only; authority exposes exactly `GET /snapshot` + `GET /reconciliation`; writes go through the Phase 2 port. | `availability-phase1-2-contract-extract.md:57-61` (§4.1) |
| P-2 | Every source reports `RESOLVED`/`UNRESOLVED` and **declares itself unresolved rather than guessing**; restriction source has six dimensions (`closedToSell, closedToArrival, closedToDeparture, minLos, maxLos, sellLimit`). | extract `:63-76` (§4.2) |
| P-3 | **Fail-closed propagation:** unresolved/blocked ⇒ `physicalAvailable: 0`, `sellableAvailable: 0` (conservative floor, *not* zero-with-certainty), `bookingEligibility: 'UNKNOWN'`/`'BLOCKED'`; **a caller may not treat `UNKNOWN` as `ELIGIBLE`.** | extract `:78-95` (§4.3) |
| P-4 | Capacity arithmetic: all floors `max(0,…)`; **`sellLimit` only ever tightens** capacity; overage measured, not forbidden. | extract `:97-117` (§4.4) |
| P-5 | Assertion failure semantics: unresolved upstream ⇒ persisted `REJECTED` with `UNRESOLVED_CAPACITY`; **fail-closed is the default, not an error path** — "when capacity cannot be safely resolved, nothing commits." | extract `:217-232` (§5.7) |
| P-6 | Availability mutation is **synchronous and fail-closed** inside the reservation transaction; no outbox/saga may substitute. | `availability-phase3-domain-specification.md:345` |
| P-7 | Unresolved capacity ⇒ no new consuming commits; **"never assume zero or available."** | Phase 3 spec `:403` |
| P-8 | **Cancellation, not deletion, is the lifecycle operation**; *no availability release may be achieved by deleting a reservation* — release only through the exact active reservation-linked assertion; hard-delete only for terminal statuses (Decision 12). | Phase 3 spec `:141` |
| P-9 | **`CHECKED_OUT` does not release its assertion** (ratified invariant). | Phase 3 spec `:180,:190`; audit §13.1 |
| P-10 | Unrecognised/unmapped status is fail-closed (`UNRESOLVED` classification). | Phase 3 spec `:191` |
| P-11 | Exactly four fact kinds; **only their combination into sellable availability is single-sourced, and only by Availability.** | `11_TARGET_BUSINESS_RULES.md:16` (TR-1.1) |
| P-12 | **Availability is the sole source of the sellable number**; no second inventory engine; no independent "available" number anywhere (incl. rebuilt A3). | TR-1.2 (`:17`), TR-10.1 (`:122`), INV-18 (`13_…:610`), `10_DECISION_RESOLUTION.md:114` |
| P-13 | Eligibility filters live in one place (TR-10.2); **stale-by-design prohibited** (TR-10.3); provenance + `UNRESOLVED` flagging are part of the contract (TR-10.4). | `11_…:123-125` |
| P-14 | **Stop sale = allotment-scoped selling permission**: never changes quantity, never releases; display remaining 0 while applied; evaluation order stop-sale-before-remaining; per hotel+date+category. | TR-7.1–7.5 (`11_…:89-93`), INV-17 (`13_…:609`) |
| P-15 | **Hotel isolation on every read/write incl. raw SQL**; no bare-id mutation; no cross-hotel availability aggregation; multi-property authorization does not exist and all rules are single-hotel. | TR-14.1–14.5 (`11_…:166-170`), INV-1 (`13_…:593`), TR-14.4 (`:169`) |
| P-16 | Legacy GBA tables and legacy `availability` counters: retained, **read-only, never authoritative**; legacy deletion/migration = Phase 11. | TR-15.1 (`11_…:176`), `13_…:718-736` (§18) |
| P-17 | Old and new availability computations **do not coexist**: derived/relabelled views only (A3 rebuild = Option A; interim label `"derived view, not sellable availability"`). | TR-15.5 (`11_…:180`), D-6/D-6a (`10_…:87-125`), plan T-38 (`14_…:563`) |
| P-18 | A3 matrix **response shape is preserved**; its values come from the authority; **availability numbers only** move — A3 restriction writes stay A3's own (plan wording, now amended by DS-04). | plan T-38 (`14_…:563`) |
| P-19 | External channel/CRS push contracts and multi-property contracts are **out of Phase 4 scope** (deferred to Phase 5+ / later). | `13_…:65` (§1.2) |
| P-20 | Analytics computation decisions (real-time vs batch) are **out of scope** (backlog). | `13_…:67` (§1.2) |
| P-21 | Rollout flags default OFF, are env `FEATURE_*`, and follow the §11 activation sequence; **`gba.consumers.cascade` MUST be ON at deploy**; wash is blocked by Deviations A+B. | plan §11 (`14_…:923-941`), `17_…:80`, `17_…:64-72` |
| P-22 | Phase 4 test evidence: 178 API suites / 1470 pass / 1 skip / 6 fail with DB env; documented baselines API 2 suites / 6 tests and web 1 suite / 10 tests. | `17_…:45-47`, `17_…:57-60` |

---

## 5. Method: Evaluation Protocol

For each DS item:

1. **Restate the question** precisely and split it into sub-questions where it mixes independent decisions (done for DS-01 in §7, DS-11 and DS-12 in §18/§19).
2. **Gather evidence** — ratified corpus first, then file:line implementation evidence. Evidence for critical claims was re-verified against source in Stage 2 (§6.2).
3. **Classify against the ratified corpus**: if a ratified rule answers it → `PRESERVED FROM PRIOR PHASE`; if a prior decision is altered → `AMENDED DECISION` naming the superseded text; otherwise → `NEW DECISION` under this protocol; if evidence is insufficient → `UNRESOLVED`.
4. **Build the option matrix** (§2.2), minimum three options including status quo.
5. **Select** with written justification; no numeric scoring; rejected options keep their reason.
6. **Write the business rule(s)** in the §2.3 format — input, condition, source, calculation, output, error behavior, consumer duty, prohibited behavior — testable without interpretation.
7. **Close**: assign blocker class (§21), dependencies (§22), traceability row (§23), and the Stage 3 requirement.

**Hard prohibitions applied throughout (from the Stage 2 brief):**
- No restriction *policy* is invented: no precedence ranking, no invented closure/LOS semantics, no new sellable formula.
- Ratified rules are not re-opened for convenience; "legacy works this way" alone is never sufficient.
- No fail-open shortcut may be selected where a ratified rule requires fail-closed.
- Compatibility with a broken consumer is never, by itself, a reason to keep a competing number.
- Unresolved items are declared `UNRESOLVED — EVIDENCE REQUIRED` with the exact missing evidence, not defaulted.

---

## 6. Evidence Base from Stage 1

### 6.1 Findings and decision items (as received)

- **Findings:** 27 (F-01…F-27; 4 critical, 9 high, 10 medium, 4 low) — audit §10.
- **Decision items:** 12 (DS-01…DS-12) — audit §12.
- **Stage 1 blockers declared:** 1 hard (DS-01), 1 conditional (DS-02) — audit §16.
- **Risk register:** R-01…R-13 — audit §11 (dispositions in §23/§24).

### 6.2 Stage 2 re-verification (read-only; performed for this document)

| Claim | Re-verified evidence |
|---|---|
| Production DI binds the unresolved stub | `apps/api/src/modules/availability/availability.module.ts:26` → `{ provide: RESTRICTION_EVALUATOR, useClass: UnresolvedRestrictionAdapter }`; `unresolved-restriction.adapter.ts:47-59` hardcodes `status:'UNRESOLVED'` and always-populated `unresolvedSources`; it *does* partially evaluate `rate_restrictions` via `crs.evaluateRestrictions` (`:17-35`) but still marks the outcome and dimensions `UNRESOLVED` (`:47-59`). |
| Snapshot propagation | `availability-snapshot.service.ts:104-107`: `sellableAvailable: !sourceUnresolved && restriction.status==='RESOLVED' && !restrictionBlocked ? calculation.sellableAvailable : 0`; `bookingEligibility: sourceUnresolved || restriction.status==='UNRESOLVED' ? 'UNKNOWN' : (restrictionBlocked ? 'BLOCKED' : 'ELIGIBLE')`. |
| Assertion fail-closed | `availability-assertion.service.ts:881-887` `unassertableReason` → `UNRESOLVED_CAPACITY` whenever `bookingEligibility==='UNKNOWN'` **or** `restrictionOutcome.status==='UNRESOLVED'` **or** `unresolvedSources.length`; called from live `getSnapshot` (`:96`, `:286`, `:467`). |
| Rejection surfaces as write failure | `reservation-availability-wiring.ts:76-81` throws `AppError('AVAILABILITY_ASSERTION_REJECTED', …, 409)` when result ≠ `ACTIVE`. |
| Reservation create path is wired | `create-reservation.handler.ts:9` (`buildWriteIdentity`) → repository create asserts via port (`reservation.repository.ts:560-576`). |
| CRS `book` bypasses the authority | `crs-engine.service.ts:343-415`: legacy `inventoryDomain.assertAvailability` + `reserve` + raw `rate_restrictions` CTA/CTD check, then `reservation-persistence.domain-service.ts:69-79` raw `INSERT INTO reservations … 'CONFIRMED'` — no assertion port. |
| Dual-write on modify | `reservation.repository.ts:621` (`replaceReservationAssertion`) then `:625` (`crs.modifyReservation`); status changes use `applyStatusAvailability` only (`:646`, `:973`, `:1000-1055`). |
| Legacy FO gate | `upgrade-room.handler.ts:69` + `inventory.domain-service.ts:121-126` (`isAvailable` returns `true` when the row is absent). |
| Unscoped legacy read | `rates-inventory.service.ts:34-39` (counts without `hotel_id`, `occupancyPct = round((occupied/total)*100)`). |
| A3 restriction writes exist; second restrictions table is unread | `availability-sales.controller.ts:676-707` (CTA/CTD/zero-sell upserts), `:825-885` (sell-limit, min/max LOS, CTA/CTD, zero-sell, `restrictions` `rate_code='CUTOFF'`); repo-wide search finds **no reader** of `restrictions` other than its Prisma model. |
| A3 combines selling permission outside the authority even when flag ON | `availability-sales.controller.ts:384-405` (raw restriction reads) + `:540-547` (`available: hasZeroSell ? 0 : authority?.available ?? 0`). |
| Authority physical inputs | `availability-source.adapter.ts:59-90`: `rooms` + `out_of_order` + `out_of_service` (+ `overbooking_limits`, GBA, allotment, reservations) — **`room_inventory` is not an authority input**. |
| Conflict handling precedent inside the authority | `availability-source.adapter.ts:129-137` (`stopSaleConflicts` ⇒ `allotmentUnresolvedReasons` ⇒ UNRESOLVED promotion) — the authority already declares conflicts unresolved rather than choosing. |
| No in-repo writer for several input tables | Repo-wide search (apps + packages, incl. raw SQL) finds **no writer** for `rate_restrictions`, `out_of_order`, `out_of_service`, `room_inventory`. A3 writes only the restriction tables listed above. Housekeeping `FLAG_OOO` is a discrepancy-resolution label (`person-discrepancy.ts:2`), not an `out_of_order` writer. |
| Delete path has no availability operation; availability links are `Restrict` | `reservation.repository.ts:889-923` (terminal-only guard + child deletes + `reservations.delete`, no availability op); schema `reservation_availability_state` and `reservation_availability_operations` both reference `reservations` with `onDelete: Restrict` (`schema.prisma:17399`, `:17426`); `availability_assertion_balances` is keyed `(hotel_id, room_type, stay_date)` with **no reservation FK** (`:17316-17328`); movements carry free-text `reference_id` (`:17375`). |
| Broken frontend calls | `reservation.api.ts:405,420` → `/availability/matrix/logs`; `:431` → `/activities/availability/interval-update` with `ratePlanCodes/startDate/endDate`; backend declares `@Get/@Post('availability/logs')` (`:719/:752`) and `@Post('availability/interval-update')` (`:773`, expects `ratePlans`/`dateRange`); `POST availability/bulk-update` (`:651`) already matches the frontend (`reservation.api.ts:247`). |
| Matrix label exists today | `availability-sales.controller.ts:612` → `...(authoritative ? {} : { label: 'derived view, not sellable availability' })`. |
| Flags | 7 env `FEATURE_*` flags, all default OFF, read via `ConfigService.getFeatureFlag`; root `.env.example` and `compose.yaml` contain **no** `FEATURE_` entries; platform DB flag system (`platform/configuration/feature-flag.service.ts`) has no availability usage. |
| Test harness gating | `availability-postgres.harness.ts:8-9` (`describe.skip` without `AVAILABILITY_TEST_DATABASE_URL`); Phase 4 executed the full battery *with* DB env (`17_…:45`). |
| Zero consumers of the authority read surface | audit §15.3-E1 (0 hits for `availability/snapshot|availability/reconciliation` across `apps/**`, `gateway/**`, `packages/**`). |

### 6.3 Items that remain UNVERIFIED (carried, not hidden)

These are evidence requirements, not decisions (see §30). Stage 2 executed nothing (no tests, no builds, no DB):

| # | Unverified item | Needed by |
|---|---|---|
| U-1 | Runtime confirmation of F-01 (assertion rejection under production DI). | DS-01 operational gate |
| U-2 | Runtime winner of the `GET /tax-rates` collision (F-20). | DS-11 |
| U-3 | Population source for `room_inventory`, `out_of_order`/`out_of_service`, `rate_restrictions` (external job/DBA script? none in repo). | DS-12 → DS-01 |
| U-4 | Documented test baselines (API 2 suites/6 tests, web 1 suite/10 tests) vs static analysis (F-22). | DS-09 |
| U-5 | External (out-of-repo) consumers of `/rates/engine/*`. | DS-03 |
| U-6 | Live behavior of the OTA overbooking branch (F-27). | DS-03 |

---

## 7. DS-01 First-Class Analysis — Twelve Questions

DS-01 ("restriction authority") is the single hard blocker declared in Stage 1. It is decomposed here before any option is evaluated.

**Q1 — Does any ratified Phase 1–4 rule address restriction resolution?**
*Answer:* It is addressed at three layers, but never as a combination policy:
- Phase 1 defines the **Restrictions source contract**: six dimensions, each `RESOLVED`/`UNRESOLVED`, "declares itself unresolved with a reason rather than guessing" (extract `:63-76`, P-2).
- Phase 4 TR-1.1 names **selling permission (stop sale, restrictions)** as one of exactly four fact kinds, and states that **only their combination into sellable is single-sourced, by the authority** (`11_…:16`, P-11); TR-10.1/TR-10.2 make that authority the only eligible filter owner (P-12/P-13).
- **No Phase 3 or Phase 4 rule states how individual restriction stores combine** (Phase 3 spec/decision sheet contain no restriction ruling; Phase 4 ruled only allotment stop sale, TR-7.x).
*Consequence:* the *existence* of restriction evaluation inside the authority is ratified; the *precedence/aggregation policy* is not.

**Q2 — What does `UNRESOLVED` mean?**
*A:* A source **declines to assert what it cannot compute**, carrying `reasons` and `unresolvedSources` (P-2); invalid rows are flagged with provenance and never silently clamped (INV-8, `13_…:600`; TR-10.4). It is a *statement about knowledge*, not a capacity value.

**Q3 — What does `bookingEligibility: 'UNKNOWN'` oblige callers to do?**
*A:* They may not treat it as `ELIGIBLE` (P-3). Downstream, an assertion against `UNKNOWN` must be `REJECTED` with `UNRESOLVED_CAPACITY` (P-5). Any consumer that displays it must distinguish it from `ELIGIBLE` (see BR-5-011).

**Q4 — Is `sellableAvailable: 0` under `UNRESOLVED` a business fact?**
*A:* No. It is the **conservative floor**; "never reported as zero-with-certainty" (P-3). Rendering it to a user as a definite number is a violation of the contract, not a faithful display (BR-5-011/BR-5-012).

**Q5 — Is the permanent-`UNRESOLVED` binding a business rule or an implementation gap?**
*A:* **Implementation gap.** Evidence: the adapter's own reason string — "Existing canonical evaluator does not cover every applicable authoritative source" (`unresolved-restriction.adapter.ts:47-59`) — declares incompleteness; TR-1.1 requires selling permission to be combined by the authority; the D-6 decision text says no other component may combine it (`10_…:114`). Keeping the stub forever would make the authority permanently unusable while *claiming* conformance — neither outcome is authorized by any ratified rule.

**Q6 — Which stores are in scope for the evaluator (input inventory)?**
*A:* Derived from evidence, not invention (§19, BR-5-001): every store with an in-repo writer (`close_to_arrival`, `close_to_departure`, `zero_sell_limits`, `room_type_sell_limits`, `minimum_length_of_stay`, `maximum_length_of_stay` — all written by A3), plus `rate_restrictions` (read by the CRS booking gate and the adapter), plus the **ratified** allotment stop-sale inputs (`allotment_daily_quotas.stop_sale_active`, `allotment_stop_sales` — already resolved by the source adapter). Excluded pending evidence: `restrictions` (`rate_code='CUTOFF'` — written by A3, read by nothing) and `channel_restrictions` (no writer, no reader). Population evidence for all input stores is U-3 → DS-12.

**Q7 — Does the authority already evaluate any restrictions today?**
*A:* Yes, partially: `rate_restrictions` CTA/CTD/Min/Max via `crs.evaluateRestrictions` (`unresolved-restriction.adapter.ts:17-35`, `crs-engine.service.ts:138-163`) and allotment stop sale via the source adapter (`availability-source.adapter.ts:129-145`, snapshot override `availability-snapshot.service.ts:58-72`) — but the outcome is forced `UNRESOLVED` regardless.

**Q8 — Is *blocking* semantics for resolved restrictions ratified?**
*A:* Yes. `restrictionBlocked` = `closedToSell`/`closedToArrival`/`closedToDeparture` true, or stay length outside `minLos`/`maxLos` ⇒ `bookingEligibility: 'BLOCKED'` (extract `:93-95`, P-3). Stop-sale-for-allotments additionally has TR-7.x (P-14).

**Q9 — Is `sellLimit` arithmetic ratified?**
*A:* Yes: `sellableCapacity = min(capacityWithOverbooking, max(0, sellLimit))` — **can only tighten** (extract `:105-112`, P-4).

**Q10 — What happens when two applicable stores disagree on a dimension?**
*A:* **No precedence ranking exists anywhere in the corpus, and inventing one is prohibited.** What *is* ratified is the fallback: cannot-determine ⇒ declare `UNRESOLVED` with provenance (P-2, INV-8, TR-10.4), which the authority already does for allotment stop-sale conflicts (`availability-source.adapter.ts:129-137`). Therefore: conflict ⇒ dimension `UNRESOLVED` + `sourceConflicts` populated (BR-5-003). Any *explicit precedence ranking* remains `UNRESOLVED — EVIDENCE REQUIRED` (needs product/operational evidence, §30 E-2).

**Q11 — May assertion enforcement be relaxed (fail-open) so writes work while restrictions are unresolved?**
*A:* **No.** That would contradict P-3/P-5/P-6/P-7 directly. Fail-open is not an available option (see option C in §8); the only compliant path is to make resolution possible.

**Q12 — What evidence must be obtained before the outcome can be declared operational?**
*A:* U-1 (runtime confirmation of the production DI chain) + U-3/DS-12 (input population evidence) + the evaluator implementation itself. Until all three close, `gba.a3.authoritative` stays OFF and no consumer may be cut over onto authority numbers (§21 BLK-P5-01).

---

## 8. DS-01 — Decision, Options, and Testable Rules

**Question (restated):** must the authority evaluate restriction/selling-permission stores (thereby becoming able to return `ELIGIBLE` and support enforceable assertions), or does fail-closed remain permanent behind a documented interim rule?

**Primary status:** `NEW DECISION`
**Blocker class:** `HARD` until the sub-items in §8.2 close (see §21 BLK-P5-01).

### 8.1 Option matrix

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | **Build the authoritative `RestrictionEvaluator`** (scope per BR-5-001), keeping fail-closed semantics unchanged | Implement resolution over the evidenced store set; `UNRESOLVED` only when a store is actually indeterminate/conflicting | Aligned: TR-1.1 (P-11), TR-10.1/10.2 (P-12/P-13), Phase 1 source contract (P-2); does not change any formula (P-3/P-4 preserved) | Authority can return `ELIGIBLE`/`BLOCKED` honestly; assertions become meaningful; UI can show real sellable | Unlocks DS-02 cutover, DS-03 booking re-point, flag `gba.a3.authoritative` | Largest work item; reversible (stub binding remains available behind a flag); no schema change | **SELECTED** |
| B | **Keep the stub permanently; document "authority is read-only/fail-closed forever"** | Assertions always reject; snapshot always `0/UNKNOWN`; flags stay OFF | Contradicts TR-1.1/TR-10.1 (a fact kind that is never combined) and the D-6 text (`10_…:114`); technically "compliant" with P-3 but defeats the ratified purpose | No usable sellable number ever; guest-facing availability cannot migrate; Phase 5 has no deliverable | Product permanently runs two legacy engines | Unbounded: entrenches F-01/F-02/F-03 indefinitely | **REJECTED** — preserves the defect, not a rule |
| C | **Relax fail-closed** (treat `UNKNOWN` as `ELIGIBLE`, or suppress `unresolvedSources` so assertions pass) | Make writes succeed by not reporting uncertainty | **Directly violates** P-3, P-5, P-6, P-7 (extract `:93`, Phase 3 `:403`, `:345`) | Oversell exposure: bookings confirmed against uncomputable capacity; `UNKNOWN` silently becomes permission | Looks green immediately | Catastrophic and non-reversible once bookings exist | **REJECTED** — prohibited by ratified rules |
| D | **Scope-gated interim**: allow only dimensions with ratified semantics to be reported, everything else stays `UNRESOLVED` | Partial resolution (e.g. stop-sale only) | Compliant in the narrow sense, but `RestrictionOutcome.status` is per-outcome: any open dimension keeps the whole stay `UNRESOLVED` (P-2) ⇒ identical runtime behavior to option B until the full evidenced scope is covered | No partial unlock in practice | None | Low value | **REJECTED as a standalone** — its store-scope discipline is absorbed into A (BR-5-001) |

**Decision:** **Option A**, with the interim rules in §8.3 binding from this document onward.

### 8.2 Sub-decision status table

| Sub-item | Question | Status | Notes |
|---|---|---|---|
| DS-01.1 | Semantics of `UNRESOLVED` / `UNKNOWN` / `0` | `PRESERVED FROM PRIOR PHASE` | P-2, P-3, P-5, P-7 |
| DS-01.2 | Authority must evaluate selling permission (ship a real evaluator) | `NEW DECISION` (Option A) | BR-5-001, BR-5-004, BR-5-008 |
| DS-01.3 | Handling of indeterminate/conflicting inputs | `NEW DECISION` — conflict ⇒ `UNRESOLVED` + provenance; **explicit precedence ranking `UNRESOLVED — EVIDENCE REQUIRED`** | BR-5-003 (application of P-2/INV-8/TR-10.4; the existing stop-sale conflict pattern is the implementation precedent) |
| DS-01.4 | Store applicability / absence semantics | `NEW DECISION` — no row ⇒ that store imposes nothing, recorded with provenance | BR-5-002; evidence: two independent legacy implementations agree (`crs-engine.service.ts:163` returns `cta:false/minLos:null` on no row; A3 set-membership `availability-sales.controller.ts:430-447`) |
| DS-01.5 | Input population evidence (which stores actually carry data, and who writes them) | `UNRESOLVED — EVIDENCE REQUIRED` (U-3) → feeds BR-5-001 scope confirmation and BR-5-033 | §30 E-1 |
| DS-01.6 | Runtime confirmation of the production chain (U-1) | `UNRESOLVED — EVIDENCE REQUIRED` | §30 E-3; gates operational enablement, not the rule |
| DS-01.7 | May assertions be relaxed to make writes work? | `PRESERVED FROM PRIOR PHASE` — **no** | P-3/P-5/P-6/P-7 |

### 8.3 Interim rules binding now (before the evaluator ships)

- `gba.a3.authoritative` **must remain OFF** (BR-5-009). Turning it ON while the stub is bound would zero every matrix cell (`availability-sales.controller.ts:540`).
- The authority read surface **may** be consumed structurally (hooks, types, wiring) but any surface that renders numbers must apply BR-5-011 (eligibility surfacing) so zeros are never presented as certainty.
- No consumer may be re-pointed onto the authority for *write gating* until U-1 and the evaluator exist (this is the ordering behind DS-03/DS-05).
- The stub binding stays available as a rollback artifact; removing it is not required by this decision.

---

## 9. DS-02 — Frontend Read Contract

**Question:** do consumers migrate onto `GET /properties/:propertyId/availability/snapshot`, or does `GET /availability/matrix` remain the wire contract (authority-backed) with the snapshot hidden behind it — and how are `label` / `bookingEligibility` / roomType×bedType grouping handled?

**Status:** `NEW DECISION` (Phase 4 decided the *server-side* matrix projection — T-38/D-6a — but never chose the frontend contract).
**Blocker class:** was `CONDITIONAL` on DS-01 → **released as a decision**; execution remains gated by BLK-P5-01.

### 9.1 Option matrix

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | Frontend migrates **directly to `/snapshot`**; matrix retired | One contract, full provenance per date | Aligned with P-1/P-12; **contradicts** ratified T-38 "response shape preserved" (P-18) and would discard the sanctioned interim label | Single honest contract | Rewrites every grid/table consumer at once | High blast radius (availability page, quick-book, rates tab) | **REJECTED** — discards a ratified preservation |
| B | Frontend stays on `/matrix` only; snapshot stays server-side | No new hook; matrix carries authority numbers | Aligned with P-17/P-18; but matrix cells carry **no** eligibility/provenance (`t38-authoritative-matrix.spec.ts:14-25` CELL_KEYS) ⇒ consumers cannot meet P-3 duty (no `UNKNOWN`-as-`ELIGIBLE`) without extra data | Cannot surface eligibility honestly | Keeps one call | Low risk, but forces ad-hoc fields later | **REJECTED** as the *only* contract |
| C | **Two sanctioned contracts**: `/snapshot` = canonical domain read (provenance + eligibility); `/matrix` = sanctioned **projection** for grid surfaces (shape preserved, values from authority, interim label while OFF); additive eligibility fields permitted | Combines A's honesty with P-17/P-18 | Both obligations satisfiable: numbers single-sourced (P-12), eligibility surfaced (P-3), shape preserved (P-18) | Users see the same grid they know; new surfaces (admin/mobile greenfield) use the canonical read | Moderate; projection can be migrated independently of the page | **SELECTED** |

**Decision:** **Option C.**

### 9.2 Rules

- **BR-5-010** — sanctioned read contracts (§20).
- **BR-5-011** — eligibility must be surfaced; `UNKNOWN` never rendered as a definite number.
- **BR-5-012** — no client-side availability computation; presentation-only transforms; bed-type split is a partition of the authority number.
- **BR-5-013** — one shared contract type for the snapshot payload; a projection type for matrix; no hand-rolled duplicates (F-13).
- **BR-5-014** — `/reconciliation` remains the sanctioned legacy-vs-authority comparison, read-only (P-16).

**Bed-type grouping decision (NEW, evidence-based):** matrix cells are keyed `roomType|bedType` and today distribute room-type-level authority numbers proportionally across bed types (`availability-sales.controller.ts:449-466`, `:565-586`). Ratified rules only prohibit a second *available number* (P-12), not a partition of one. **Selected:** keep the split as display-only, subject to BR-5-012's conservation law (bed-type cells for a room type must sum exactly to the authority figure for that room type and date; rounding residue must be assigned deterministically and disclosed in the projection, never dropped). Rejected alternative: dropping bed-type granularity (breaks the existing grid and quick-book layout without business justification).

---

## 10. DS-03 — Disposition of Legacy Rates/CRS Availability

**Question:** retire `/rates/engine/*` and `GET /rates/availability`, or re-point them at the authority? (Includes the quick-book write gate, the FO upgrade gate, and tenant-scoping remediation.)

**Status:** `NEW DECISION`.
**Blocker class:** `CONDITIONAL` (ordering: retirement/re-point of *booking-gated* paths depends on BLK-P5-01; read-side retirement does not).

### 10.1 Option matrix

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | **Retire the availability-bearing surfaces; keep pricing** | Remove/strip availability outputs and gates; pricing/quote mechanics stay | Aligned: INV-18 (P-12), TR-15.5 (P-17), Phase 3 P-6 (every reservation mutation through the authority); F-18 precedent (route retirement was executed in Phase 4) | One availability truth; oversell gate moves to the authority | Booking flow changes (quick-book, FO upgrade) once authority is live | Route removal reversible by redeploy; requires UI re-point in same change | **SELECTED** |
| B | Re-point legacy endpoints to the authority behind the same URLs | Keep URLs, change the number | Keeps dead URLs alive; `/rates/engine/book` cannot "re-point" without becoming the reservation create path (it raw-inserts `CONFIRMED` rows) | Same truth, worse surface | No URL churn | Hides the migration; preserves legacy request shapes indefinitely | **REJECTED** — compatibility-only argument (prohibited by §5) |
| C | Do nothing until Phase 11 | Legacy stays authoritative for those screens | Violates INV-18 now, and leaves the unscoped `GET /rates/availability` (F-06, R-05 tenant exposure) live | Two truths persist through Phase 5 | None | Cross-tenant read exposure persists | **REJECTED** |

**Decision:** **Option A**, endpoint-by-endpoint below.

### 10.2 Endpoint dispositions (binding)

| Endpoint | Disposition | Rule |
|---|---|---|
| `GET /rates/availability` | **RETIRE** (or reduce to a payload containing no availability/occupancy figure). Any retained count must be hotel-scoped. | BR-5-015 |
| `GET /rates/engine/quote` (its `available` / `blockReasons` signal) | **Re-point the availability signal to authority eligibility**; pricing/hash mechanics retained. | BR-5-016 |
| `GET /rates/engine/availability`, `GET /rates/engine/restrictions` | **RETIRE** as availability sources (restrictions display moves to the authority projection, DS-04). | BR-5-016 |
| `POST /rates/engine/book` | **RETIRE as a booking path.** Booking must execute through the reservation create command with assertion (P-6). | BR-5-017 |
| `POST /rates/engine/modify` | Governed by DS-05 (its legacy leg is the dual-write leg). | BR-5-022 |
| `POST /rates/engine/release` | **RETIRE** once no booking path writes legacy counters: a legacy counter decrement is not an availability release (P-12/P-16). | BR-5-018 |
| FO upgrade gate `upgrade-room.handler.ts:69` | **Re-point to authority eligibility**; the legacy `isAvailable` absent-row `true` default (`inventory.domain-service.ts:126`) must never gate a write. | BR-5-019 |
| `availability` table (legacy counters) | Table retained (P-16, Phase 11); its **last writer disappears when the last CRS booking path is re-pointed**; no new readers. | BR-5-018 |

**Note on U-5 (external consumers of `/rates/engine/*`):** retirement must be preceded by an external-consumer check; if out-of-repo consumers exist, they are recorded and given a deprecation window — this does not change the disposition, only its schedule (§30 E-5).

**Note on F-27 (OTA overbooking branch):** `webhook.service.ts:104` matches `err.message.includes('inventory')` while the authority throws `AVAILABILITY_ASSERTION_REJECTED` — the branch is governed by BR-5-020: error detection must use the **deterministic error contract**, not message substrings. Live behavior is U-6 (evidence, §30 E-6).

---

## 11. DS-04 — Restriction Write Ownership

**Question:** keep raw-SQL restriction writes in Activities (Phase 3 L-12 "KEEP — Phase 1 authority"), or move them onto an authority-owned write path?

**Status:** `NEW DECISION` (with one `UNRESOLVED` sub-item).
**Blocker class:** `CONDITIONAL` (BLK-P5-02: the DS-11 UI repair must not land on raw legacy writes before this path exists).

### 11.1 Option matrix

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | **Authority-owned restriction write path** (hotel-scoped, validated, invalidating, audited); A3 becomes a caller | Writes move behind the authority; A3 keeps its UI | Aligned: TR-1.1 (P-11) puts combination in the authority; TR-10.2 (P-13) puts filters in one place; TR-10.4 provenance; INV-1 hotel scoping (today's raw SQL is scoped but unvalidated) | Selling permission becomes a first-class authority input with provenance — prerequisite for DS-01's evaluator to be meaningful | Staff use the same screens; write semantics unchanged | Moderate; rollback = keep old path behind flag | **SELECTED** |
| B | Keep raw writes; only remove A3's `available` overlay | Smallest change | Removes the second *combination* (INV-18) but leaves unvalidated writes, F-26 SQL-interpolation surface, and no invalidation | Selling permission edited outside the authority's audit | None | Leaves write/read asymmetry | **REJECTED** as final state (acceptable only as a temporary stepping stone inside A) |
| C | Keep everything (status quo) | Writes + overlay stay | Violates INV-18 (`:610`) directly — `available: hasZeroSell ? 0 : authority.available` is a second combination | Contradictory numbers persist | None | Permanent non-conformance | **REJECTED** |

**Decision:** **Option A.** Phase 3 L-12 "KEEP" referred to *retaining the operational capability* while Phase 1 authority consumed it; it does not authorize a second combination engine (P-12 supersedes any reading that would).

### 11.2 Rules and open sub-item

- **BR-5-021** — restriction writes go through the authority-owned write path (scoped, validated, invalidating, provenance/audit); afterwards, raw `$executeRawUnsafe` writes to restriction tables are prohibited (also closes F-26's write-side surface).
- **BR-5-022** — restriction reads for display come from the authority projection (removes raw reads at `:384-405`).
- **BR-5-023** — A3's availability overlay is removed; combination happens only inside A1 (BR-5-008).
- **DS-04 sub-item `restrictions` (`rate_code='CUTOFF'`):** written by A3 (`:878-885`), **read by nothing** (verified §6.2). Its business meaning is not defined anywhere in the corpus. Status: `UNRESOLVED — EVIDENCE REQUIRED` (§30 E-4). Interim rule: the writer must not be migrated into the new path until its meaning is evidenced — i.e., it is **not** silently given authority status (BR-5-024).

---

## 12. DS-05 — Reservation Write-Path Convergence (Dual-Write Ordering)

**Question:** is the legacy leg `reservation.repository.ts:625` removed in Phase 5, or deferred to Phase 11 legacy retirement? (Extended in Stage 2 to the related write-identity gaps F-16.)

**Status:** `NEW DECISION`.
**Blocker class:** `CONDITIONAL` (ordering dependency, §22).

### 12.1 Option matrix

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | **Remove the legacy leg in Phase 5, after its readers are disposed** | Ordering: DS-01 live → DS-03 dispositions land → then remove `:625` | Aligned: TR-15.5 "old and new computations do not coexist" (P-17); plan §11 "no dual-write" (single-writer principle); F-04 shows the two stores drift *by design* | Ends drift-by-design; legacy counters stop being a second write truth | None visible to staff (legacy counters already have no authoritative reader after DS-03) | Reversible (leg is a single call) | **SELECTED** |
| B | Remove immediately in Stage 3 | Cut the leg first | While legacy readers still exist (rates tab, upgrade gate, CRS book), removing the writer silently freezes what they see | Risk of misleading displays during transition | Irreversible-ish (behavior change without consumers moved) | **REJECTED** — wrong order |
| C | Defer entirely to Phase 11 | Keep dual-write until legacy retirement | Phase 11 defers *table deletion* (P-16), not continued divergence; TR-15.5 forbids coexisting computations during transition | Two stores keep disagreeing through Phase 5; R-06 persists | None | **REJECTED** — over-reads the Phase 11 deferral |

**Decision:** **Option A**, with explicit ordering: **DS-01 operational → DS-03 endpoint dispositions → DS-05 removal** (§22 D-3/D-4). Until then the status quo (both writes) is *preserved deliberately* (BR-5-026), because removing the writer first would strand legacy readers.

### 12.2 Rules (incl. F-16 write identity)

- **BR-5-025** — single write truth per fact: after the ordering gates, the reservation modify path writes availability only through the assertion port; `crs.modifyReservation` is no longer invoked for availability counters.
- **BR-5-026** — while the authority is not yet operational: no write-path reduction (status quo preserved; no partial cuts).
- **BR-5-027** (preservation of Phase 2 §5.4, extract `:177-185`; disposition of F-16) — every availability-bearing write carries a deterministic operation identity: explicit `idempotencyKey` where the caller has one, never `hotelId || 'default'` (the authority rejects `default`, `availability-snapshot.service.ts:19`), and never an absent write identity that silently mints `randomUUID()` operation keys (`change-rate.handler.ts:30`). *This is a restatement of a ratified rule, not a new one*; its Stage 3 consequence is implementation, not decision.

---

## 13. DS-06 — Canonical Occupancy / KPI Definition

**Question:** which number do dashboards, analytics, FO KPIs, and command-center widgets publish — snapshot-derived occupancy, an analytics contract, or a documented "reporting-only" exemption?

**Status:** composite — availability-labelled numbers `RESOLVED` (via preservation); the *occupancy analytics contract* is `PRESERVED FROM PRIOR PHASE` as **deferred** (P-20: analytics computation decisions are out of scope).
**Blocker class:** `NON-BLOCKING` / `DEFERRED`.

### 13.1 Option matrix

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | **Split the question**: any *available/sellable* figure ⇒ authority only; *occupancy* KPIs ⇒ declared reporting, source-labeled, hotel-scoped, never derived from legacy availability | Two clearly bounded rules | Aligned: INV-18/P-12 covers "available" numbers; Phase 4 §1.2 defers analytics computation (P-20); Phase 4 fact-ownership puts room-state facts outside Availability (`13_…:139-156`); audit §13.9 confirms room-level FO facts are out of authority scope | Ends contradictory "avail" labels (F-03/F-12) without re-deciding analytics | Dashboards keep working; labels/sources become honest | Easily reversible | **SELECTED** |
| B | Mandate occupancy be computed from the snapshot now | One formula everywhere | Would decide analytics architecture that Phase 4 explicitly deferred (P-20); occupancy ≠ sellable (occupied/total uses different facts) | Premature contract | Restructures analytics | Hard to undo if wrong | **REJECTED** — exceeds Phase 5 authority |
| C | Exempt everything as "reporting" | No rules | Violates INV-18 for figures labelled *available* | Contradictions persist | None | Permanent ambiguity | **REJECTED** |

**Decision:** **Option A.** The occupancy analytics *contract* itself remains `DEFERRED` (status recorded, not dropped).

### 13.2 Rules

- **BR-5-028** — any figure labelled available/sellable (including headers, KPI tiles, tab totals) derives from the authority; mislabelled computations (e.g. `AvailabilityPage.tsx:1006-1015` summing `physicalInventory` under "avail") are removed or re-labelled to what they actually compute.
- **BR-5-029** — occupancy KPIs are reporting: hotel-scoped, source-declared (room-state/reservation counts), never derived from a legacy *availability* source, never presented as availability; hardcoded/mock values (F-09/F-10/F-12) are not permitted to represent live truth.
- Deferred: the canonical occupancy formula/real-time-vs-batch contract (P-20) — carried in §31 as `DEFERRED`, with dependency note: if it later produces an availability-labelled figure, BR-5-028 governs it.

---

## 14. DS-07 — Outbound Channel/CRS Push Contract (scope)

**Status:** `PRESERVED FROM PRIOR PHASE` (ratified deferral P-19) → **`DEFERRED`, OUT OF PHASE 5 SCOPE**.
**Blocker class:** `DEFERRED`.

**Evidence:** `13_FINAL_DOMAIN_SPECIFICATION.md:65` defers external channel/CRS push contracts; F-15 confirms no publication path exists (`channels.service.ts:162-183` logs caller-supplied numbers; `channel_availability` never written; `channel-sync.job.ts` is a stub).

**Option matrix (abridged for a scope item):**

| # | Option | Verdict |
|---|---|---|
| A | Keep deferred; add a guard rule so nothing *pretends* to publish | **SELECTED** |
| B | Build the push contract now (B-6 boundary) | **REJECTED** — re-opens a ratified deferral without the underlying read contract being operational |
| C | Remove the logging endpoint as "dead" | **REJECTED** — out of scope for a decision stage; recorded as Stage 4 hygiene candidate |

**Rule:** **BR-5-030** — until a ratified push contract exists, no endpoint may publish an availability number from any source other than the authority; the existing caller-supplied `channel_availability_log` write is explicitly **non-authoritative** and must not feed any UI or eligibility decision.

---

## 15. DS-08 — Multi-Property Contracts (scope)

**Status:** `PRESERVED FROM PRIOR PHASE` (TR-14.3, P-15) → **`DEFERRED`, OUT OF PHASE 5 SCOPE**.
**Blocker class:** `DEFERRED`.

**Evidence:** `11_TARGET_BUSINESS_RULES.md:168` — "Cross-hotel operations … require explicit multi-property authorization that does not yet exist — all Phase 4 rules are single-hotel"; INV-1 (`13_…:593`); TR-14.4 (`:169`).

**Rules:** **BR-5-031** — every Phase 5 rule, read, write, and projection is hotel-scoped; no availability computation aggregates across hotels; **BR-5-032** — `hotelId === 'default'` (or absent property context) is rejected at every authority entry (`availability-snapshot.service.ts:18-21`). No multi-property work is added to Phase 5 scope by this document.

---

## 16. DS-09 — Test-Execution and Evidence Policy

**Question:** do later stages require executed runs (postgres battery + full suite) and a declared `AVAILABILITY_TEST_DATABASE_URL`, and how are baselines reconciled?

**Status:** `NEW DECISION` (process rule; Phase 4 established the precedent at `17_…:45-47`).
**Blocker class:** `NON-BLOCKING` for decisions, but it **gates Stage 3/4 exits**.

### 16.1 Option matrix

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | **Executed-evidence policy**: every code-changing stage runs API full suite *with* the postgres env var set, web typecheck/lint/test; skip counts and baselines reported in the stage artifact | Makes green meaningful | Aligned with Phase 4 practice (`17_…:45`); closes F-17/R-10/F-22 | Decisions rest on real behavior, not static reasoning | Requires env declaration (a config change, scheduled for Stage 4 implementation) | Trivially reversible | **SELECTED** |
| B | Keep static-only verification | No execution | Leaves 48 suites silently skippable (`availability-postgres.harness.ts:8-9`) and baselines unreconciled (F-22) | Evidence stays partial | None | False confidence | **REJECTED** |
| C | Require execution but skip env declaration | Run only when a developer remembers | Same as B in CI | None | None | **REJECTED** |

**Decision:** **Option A.**

### 16.2 Rules

- **BR-5-033** — Stage 3+ code-changing stages must record: API suite totals with `AVAILABILITY_TEST_DATABASE_URL` set (harness-gated suites must be **0 skipped** for that reason), web typecheck/lint/test, and reconciliation of the documented baselines (API 2/6, web 1/10 — `17_…:57-60`) against actuals (closes F-22).
- **BR-5-034** — declaring `AVAILABILITY_TEST_DATABASE_URL` in the repo's config documentation (and CI) is a **Stage 4 implementation requirement**, decided now (closes F-17's documentation gap).
- Stage 2 executed nothing — recorded here for audit integrity (§6.3).

---

## 17. DS-10 — Flag Governance for Phase 5

**Question:** which of the 7 flags may Phase 5 touch, which config surface documents them, and what happens to the second (platform DB) flag system?

**Status:** `NEW DECISION` (rules), `PRESERVED FROM PRIOR PHASE` for the existing rollout sequence and the `gba.consumers.cascade=ON` requirement (P-21).
**Blocker class:** `NON-BLOCKING` (governs rollout; no decision waits on it).

### 17.1 Option matrix

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | **One flag mechanism, no code-level flips, evidence-gated ON for `gba.a3.authoritative`** | All availability rollout flags stay env `FEATURE_*` via `ConfigService.getFeatureFlag`; the platform DB flag system is explicitly out of scope for availability; flags are declared in config documentation; Phase 5 code never flips a flag | Aligned: P-21 sequence; closes F-18 (dual systems + undeclared flags incl. the "MUST be ON" cascade flag) | Rollout state is auditable; no accidental authority activation | Ops runbook owns flag state (Phase 4 §11 unchanged) | Fully reversible | **SELECTED** |
| B | Move availability flags onto the platform DB system | Central UI for flags | Would create a *third* truth for flag state (availability code only reads ConfigService today) | None | Adds migration work | Hard to unwind | **REJECTED** |
| C | Let Phase 5 flip flags as part of feature work | Convenient | Violates P-21 sequencing and makes review unable to distinguish code change from rollout change | None | Fast but unauditable | **REJECTED** |

**Decision:** **Option A.**

### 17.2 Rules

- **BR-5-035** — flag inventory and mechanism: the seven `FEATURE_*` flags remain the single mechanism for availability rollout; no availability flag may be added to the platform DB flag system; flag state changes are operational actions with recorded gate evidence, never a side effect of a code change.
- **BR-5-036** — config documentation: all availability `FEATURE_*` flags (at minimum the seven, incl. `gba.consumers.cascade`) must be declared in the repository's config documentation (`compose.yaml` / `.env.example`) — Stage 4 config work, decided now.
- **BR-5-037** — `gba.a3.authoritative` may be turned ON **only** after all of: evaluator implemented (DS-01.2), input population evidence received (U-3), runtime confirmation of the production chain (U-1), parity soak green. This is the operational closure of BLK-P5-01.
- **BR-5-038** — preserved from P-21: `gba.consumers.cascade` **MUST be ON at deploy** (`17_…:80`); `gba.wash.schedulerEnabled` stays blocked by Deviations A+B; `canonicalRead` → soak → `canonicalWrite` sequence is untouched; other flags remain default OFF.
- **BR-5-039 (disposition of F-14)** — the authority is `LIVE_READ` (`availability-snapshot.service.ts:107`); the `cache.delPattern('availability:…')` no-op must not be presented as a freshness mechanism: any cache introduced later must satisfy TR-10.3 invalidation for real, and today's behavior is documented as "no availability cache exists". The cascade flag's ON requirement is unaffected (it also gates reservation→pickup cascades).

---

## 18. DS-11 — Broken UI Surfaces, Route Collision, and Reservation Delete

**Question (part 1):** repair `matrix/logs` + `interval-update` (route + payload) or remove the dead UI affordances?
**Question (part 2):** disposition of reservation **delete** with respect to a still-live assertion for terminal `CHECKED_OUT` rows?
**Question (part 3):** the `GET /tax-rates` collision (F-20).

**Status:** composite — part 1 `NEW DECISION`, part 2 `PRESERVED FROM PRIOR PHASE` + `NEW DECISION` (clarification), part 3 `UNRESOLVED` (runtime winner, U-2) with a decided disposition rule.
**Blocker class:** part 1 `CONDITIONAL` (BLK-P5-02), parts 2–3 `NON-BLOCKING`.

### 18.1 Part 1 — option matrix

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | **Backend routes are canonical; frontend conforms; wiring gated on DS-04** | Fix client paths to `/availability/logs` + `/availability/interval-update` with backend payload, but only wire the restriction-editing affordance to the new authority-owned write path (DS-04) | Aligned: backend routes are implemented and table-backed (`:719/:752/:773`); frontend paths never worked (F-08); DS-04 keeps writes inside the authority | Audit-log view and interval editing become real features without creating a second write truth | Same screens | Small; reversible | **SELECTED** |
| B | Fix frontend onto the current raw endpoints immediately | Stops the 404s today | Would cement raw legacy writes (violates BR-5-021 once issued; increases R-07 bypass traffic now) | Works sooner | None | **REJECTED** — wrong ordering |
| C | Remove the dead affordances | Delete the UI | Loses product functionality that has backing artifacts (audit-log table, restriction editing) with no evidence it is unwanted | Fewer features | Reversible by redeploy | **REJECTED** as the primary path |

**Decision:** **Option A**, plus **BR-5-040** — canonical API names decided now (backend shapes are the contract: `/availability/logs` GET+POST, `/availability/interval-update` with `ratePlans`/`dateRange`; `POST /availability/bulk-update` already matches), and **BR-5-041** — no silent 404 affordance: a control is either wired to the canonical contract or explicitly hidden with a declared status; silent breakage in the primary availability screen is not an acceptable steady state (closes R-09).

**Payload detail (from §6.2):** `reservation.api.ts:431` sends `ratePlanCodes/startDate/endDate`; backend expects `ratePlans`/`dateRange{start,end}` (`:776-796`); `pageSize` vs backend paging params (audit §7.3) must be reconciled to the backend contract under BR-5-040.

### 18.2 Part 2 — delete vs live assertion

**Evidence:** Phase 3 P-8 (cancellation is the lifecycle operation; **no release by deletion**; Decision 12 terminal-only) and P-9 (`CHECKED_OUT` does not release). Current implementation: `reservation.repository.ts:889-923` enforces terminal-only, performs **no availability operation**, and does not touch assertion balances. Schema intent: `reservation_availability_state` and `reservation_availability_operations` reference `reservations` with **`onDelete: Restrict`** (`schema.prisma:17399`, `:17426`), while `availability_assertion_balances` has no reservation FK (`:17316-17328`) and movements keep a free-text `reference_id` (`:17375`).

**Interpretation:** the schema itself encodes "the reservation row may not be deleted while availability state points at it", and balances/audit survive independently. Today this path is latent (assertions fail closed under F-01, so no state rows persist); it becomes live the moment DS-01 lands.

| # | Option | Verdict reason |
|---|---|---|
| A | **Reject delete deterministically when availability state exists** (business error, no mutation, balances/audit untouched) | **SELECTED** — matches schema intent (`Restrict`), preserves P-8 (no release by deletion), keeps single-writer discipline (Availability owns its tables; BR-5-043), avoids an ugly FK violation surfacing as a 500 |
| B | Delete the state link rows to let the row go | **REJECTED** — a reservations path writing availability-owned tables; destroys provenance of who holds capacity |
| C | Release the assertion, then delete | **REJECTED** — release-by-delete is exactly what P-8 forbids in spirit and would change availability as a side effect of deletion |

**Rules:** **BR-5-042** — delete remains terminal-only (preserved, Decision 12) and **must never release or alter availability** (balances unchanged; capacity attributable to a completed stay is retained); **BR-5-043** — delete of a reservation that has availability state rows is rejected with a deterministic business error (no partial mutation, no raw FK error), and the operator guidance is that terminal reservations with availability history are retained; assertion movements/audit rows are never cascade-deleted.

### 18.3 Part 3 — `GET /tax-rates` collision (F-20)

Status: runtime winner `UNRESOLVED` (U-2). **Rule issued:** **BR-5-044** — a route may be declared by exactly one controller; the duplicate must be resolved by naming one canonical owner (verified at runtime first, U-2) and removing the other declaration; no aliasing to preserve both. Stage 4 hygiene task, non-blocking.

---

## 19. DS-12 — Physical / OOO / Restriction Input Ownership

**Question:** who writes `room_inventory`, `out_of_order`/`out_of_service`, and the restriction tables the authority must read?

**Status:** composite — ownership rules `RESOLVED`/`PRESERVED FROM PRIOR PHASE`; **population/source evidence `UNRESOLVED — EVIDENCE REQUIRED` (U-3)**.
**Blocker class:** `HARD` (paired with DS-01.5 — see BLK-P5-01) for the evidence part only; the ownership rules themselves are `NON-BLOCKING`.

### 19.1 What is decided (evidence-based)

| Store | Role | Decision | Evidence |
|---|---|---|---|
| `rooms` (+ status exclusions) | **the** physical inventory source | `PRESERVED` — authority definition stands (`NON_INVENTORY_*`, `NON_PHYSICAL_*`, `isPhysicalRoomStatus`) | `availability-source.adapter.ts:40-49`, `:60`, `:71` |
| `out_of_order`, `out_of_service` rows | physical-capacity reduction input, **read-only** to Availability | `RESOLVED` — remain an authority input owned by room-state operations (Front Office / housekeeping), consumed read-only; **no writer exists in this repository** (U-3) | `availability-source.adapter.ts:75-84`; §6.2 |
| `rooms.room_status = 'OUT_OF_ORDER'` | second OOO channel, already inside `physicalCount` | `PRESERVED` | `:45-49`, `:60` |
| `room_inventory` | **not** an authority input | `NEW DECISION` — legacy/derived projection only; A3's `availableRooms` from it (F-11) must not be presented as availability (BR-5-045); table treated as legacy (P-16) | authority never reads it (`:59-74`); readers only in A3 (`:62`, `:249-252`); no writer |
| `close_to_arrival`, `close_to_departure`, `zero_sell_limits`, `room_type_sell_limits`, `minimum_length_of_stay`, `maximum_length_of_stay` | restriction inputs | `RESOLVED` — in evaluator scope (BR-5-001); writer today is A3 only; ownership moves per DS-04 | `:676-707`, `:825-885` |
| `rate_restrictions` | restriction input | `RESOLVED` as an input (read by the booking gate and adapter); **writer unknown** (U-3) | `crs-engine.service.ts:149-163`, `:364-378` |
| `restrictions` (`rate_code='CUTOFF'`) | — | `UNRESOLVED — EVIDENCE REQUIRED` (meaning undefined; no reader) — excluded from evaluator scope until evidenced | §6.2, DS-04 sub-item |
| `channel_restrictions` | — | Excluded until a writer exists (no writer, no reader) | `unresolved-restriction.adapter.ts:44` (contextual only) |
| `allotment_daily_quotas.stop_sale_active`, `allotment_stop_sales` | ratified selling permission | `PRESERVED` (TR-7.x) — already resolved by the authority | `availability-source.adapter.ts:65`, `:129-145` |

### 19.2 Option matrix (for the input-validity question)

| # | Option | Description | Evidence / authority alignment | Business impact | Operational impact | Risk / reversibility | Verdict |
|---|---|---|---|---|---|---|---|
| A | **Evidence-gated inputs**: an input store stays in scope only while a writer/population source is identified; if none is found, the store is either removed from the input set or its contribution is flagged with provenance (TR-10.4) — never silently trusted | No phantom certainty | Aligned: TR-10.4/INV-8 (flag, don't guess); P-2 (declare rather than guess) | The authority never claims an empty store means "no OOO/restrictions" without proof | Requires a one-time population audit (U-3) | Reversible per store | **SELECTED** |
| B | Assume absence = truth for all inputs | Simple | Violates TR-10.4 where the store's existence is unproven; would let an unpopulated `room_inventory`-style table silently claim zeros | None | Fast | **REJECTED** — silent assumption |
| C | Wait for full data lineage before any evaluation | Nothing ships | Also blocks DS-01 indefinitely | Nothing changes | None | **REJECTED** — same as status quo | 

**Decision:** **Option A.**

### 19.3 Rules

- **BR-5-045** — `room_inventory` is not an authority input; no availability figure sourced solely from it may be displayed (closes F-11).
- **BR-5-046** — input-validity rule (Option A): each authority input store must have an identified writer/population source recorded; otherwise its contribution is flagged with provenance per TR-10.4 rather than reported as confirmed fact.
- Evidence requirement **E-1** (§30): a population audit covering `room_inventory`, `out_of_order`, `out_of_service`, `rate_restrictions`, and the six A3-written restriction tables (row counts per hotel, recency, and writer identification).

---

## 20. Consolidated Business Rule Catalogue

All rules are binding for Phase 5. `Stage 3 requirement` states what Stage 3 must turn the rule into (a requirement/spec line), not how to implement it.

**§20 is the canonical text of every rule.** Sections §8–§19 may summarize or point at a rule for context; where a summary and §20 ever differ, **§20 governs**. Rules are grouped by topic, so IDs are not strictly sequential inside each subsection.

### 20.1 Restriction evaluation and the single combination (DS-01, DS-04, DS-12)

**BR-5-001 — Evaluator scope**
*Source/authority:* TR-1.1 (P-11), TR-10.1 (P-12), Phase 1 source contract (P-2).
*Input:* hotelId, roomType, stayDate, arrivalDate, departureDate, optional rateCode/channelCode.
*Condition:* for every store in the evidenced scope set — the six A3-written restriction tables + `rate_restrictions` + ratified allotment stop-sale inputs (+ `channel_restrictions` only once a writer exists).
*Source of truth:* each store read hotel-scoped; dimensions: `stopSell, cta, ctd, minLos, maxLos, sellLimit, allotmentStopSale` (+ `channelRestriction` when applicable).
*Output:* per-dimension `RESOLVED | NOT_APPLICABLE | UNRESOLVED` with `sources[]`.
*Error behavior:* store read failure ⇒ that dimension `UNRESOLVED` with reason.
*Consumer duty:* none (server-side).
*Prohibited behavior:* marking an applicable dimension `RESOLVED` when its store was not evaluated; inventing precedence between stores.

**BR-5-002 — Absence semantics**
*Source:* evidence — two independent legacy implementations agree (`crs-engine.service.ts:163`; A3 set membership `availability-sales.controller.ts:430-447`).
*Condition:* a store has **no row** for (hotel, roomType/date [, rateCode]) ⇒ that store imposes nothing for that dimension.
*Output:* dimension `RESOLVED` (value = no restriction) with `sources[]` recording the absence check.
*Prohibited behavior:* treating absence as `UNRESOLVED` when the store has an identified writer and population (BR-5-046 governs the unproven case); treating absence as permission when the store's population is unproven.

**BR-5-003 — Conflict handling**
*Source:* P-2, INV-8 (`13_…:600`), TR-10.4; precedent `availability-source.adapter.ts:129-137`.
*Condition:* two or more applicable stores disagree on a dimension value.
*Output:* that dimension `UNRESOLVED`, `sourceConflicts[]` populated with store+value pairs, provenance retained.
*Error behavior:* propagates to `RestrictionOutcome.status = 'UNRESOLVED'` ⇒ BR-5-005.
*Prohibited behavior:* silently prioritizing any store; picking the permissive or restrictive value by heuristic; any explicit precedence ranking (that remains `UNRESOLVED — EVIDENCE REQUIRED`, §30 E-2).

**BR-5-004 — Outcome status**
*Condition:* `status = 'RESOLVED'` iff every applicable dimension is `RESOLVED` or `NOT_APPLICABLE`.
*Prohibited behavior:* `RESOLVED` while any `unresolvedSources[]` entry remains.

**BR-5-005 — Fail-closed propagation (preserved, restated)**
*Source:* P-3, P-5, P-7.
*Condition:* any source or the restriction outcome is unresolved.
*Output:* `physicalAvailable = 0`, `sellableAvailable = 0` (conservative floor), `bookingEligibility = 'UNKNOWN'`, null consumption figures, `unresolvedSources` surfaced.
*Error behavior:* assertion ⇒ persisted `REJECTED` / `UNRESOLVED_CAPACITY` ⇒ `AVAILABILITY_ASSERTION_REJECTED` (409).
*Consumer duty:* may not treat `UNKNOWN` as `ELIGIBLE`; must display the unresolved state distinctly (BR-5-011).
*Prohibited behavior:* fail-open of any kind (option C of §8 is permanently rejected).

**BR-5-006 — Blocked semantics and sellLimit (preserved)**
*Source:* P-3, P-4.
*Condition:* `closedToSell|closedToArrival|closedToDeparture = true`, or stay length outside `minLos`/`maxLos`.
*Output:* `bookingEligibility = 'BLOCKED'`, `sellableAvailable = 0`.
*Calculation:* `sellableCapacity = min(capacityWithOverbooking, max(0, sellLimit))` — may only tighten.
*Prohibited behavior:* raising capacity via `sellLimit`; displaying `BLOCKED` as `UNKNOWN` or vice versa.

**BR-5-007 — Allotment stop sale (preserved)**
*Source:* P-14 (TR-7.1–7.5, INV-17).
*Condition:* stop sale applied for hotel+date+category.
*Output:* selling display remaining 0; quantity/counters untouched; lift restores display only; per-quota conflict between daily flag and applied records ⇒ `UNRESOLVED`.
*Prohibited behavior:* releasing rooms on stop sale; evaluating stop sale after remaining quantity.

**BR-5-008 — Single combination**
*Source:* P-11, P-12, INV-18.
*Condition:* producing any figure presented as available/sellable.
*Prohibited behavior:* any component outside A1 combining selling permission with quantity (explicitly: A3's `available: hasZeroSell ? 0 : authority.available`, `availability-sales.controller.ts:540-547`).

**BR-5-009 — Interim operational gate**
Until the evaluator ships and §30 E-1/E-3 close: `gba.a3.authoritative` stays OFF; no consumer is cut over for write gating; the stub remains a rollback artifact.

### 20.2 Read contracts and display (DS-02, DS-06, DS-11)

**BR-5-010 — Two sanctioned read contracts**
`GET /properties/:propertyId/availability/snapshot` = canonical domain read (per-date facts, provenance, `restrictionOutcome`, `unresolvedSources`, `bookingEligibility`, `freshness: LIVE_READ`). `GET /availability/matrix` = sanctioned projection: **shape preserved**, values sourced from the authority when `gba.a3.authoritative` is ON, interim `label: 'derived view, not sellable availability'` while OFF (preserved P-17/P-18). No third contract may publish availability.

**BR-5-011 — Eligibility surfacing**
*Input:* a displayed cell/figure.
*Condition:* `bookingEligibility ∈ {ELIGIBLE, UNKNOWN, BLOCKED}` or `unresolvedSources ≠ []`.
*Consumer duty:* render `UNKNOWN` and `BLOCKED` as distinct states; when unresolved, the zero/absence must be shown as "unresolved/unknown", never as a definite available count; the interim label must remain visible whenever numbers are not authority-backed.
*Prohibited behavior:* rendering `sellableAvailable = 0` from an unresolved outcome as a certain number; hiding the label.

**BR-5-012 — No client-side availability math; projection conservation**
*Condition:* any frontend computation of available/sellable/occupancy-labelled figures.
*Prohibited behavior:* the ≥12 client formulas (audit §5.3), incl. `totalAvail = Σ physicalInventory` mislabelled "avail" and quick-book's literal `available: 0`.
*Allowed:* presentation-only transforms — grouping, formatting, ordering — and the bed-type partition, which must **conserve** the authority room-type figure exactly per date (sum of bed-type cells = authority value; deterministic residue assignment; disclosed in the projection).

**BR-5-013 — Shared contract types**
One shared TypeScript type (in `@xylo/shared` or the designated shared package) for the snapshot payload and one for the matrix projection; consumers may not hand-roll duplicates (closes F-13).

**BR-5-014 — Reconciliation read**
`GET /properties/:pid/availability/reconciliation` is the only sanctioned legacy-vs-authority comparison; read-only; discrepancies are flagged facts, never auto-repaired (P-16, `13_…:745-751`).

**BR-5-028 — Availability-labelled figures come from the authority** (DS-06): headers, tiles, tabs, and totals labelled available/sellable derive from A1; mislabelled computations are removed or relabelled to what they actually compute.

**BR-5-029 — Occupancy KPIs are reporting** (DS-06): hotel-scoped; source declared (room-state/reservation counts); never derived from a legacy availability source; never presented as availability; mock/hardcoded values may not represent live truth.

### 20.3 Legacy disposition and write convergence (DS-03, DS-05)

**BR-5-015 — `GET /rates/availability`:** retire (or strip all availability/occupancy fields); any retained count is hotel-scoped (TR-14.1).

**BR-5-016 — `/rates/engine` reads:** availability/restriction signals re-pointed to the authority; pricing and quote-hash mechanics retained.

**BR-5-017 — `/rates/engine/book`:** retired as a booking path; booking executes through the reservation create command with assertion (P-6); the raw `INSERT … 'CONFIRMED'` path may not remain a booking entry point.

**BR-5-018 — Legacy counters:** no new readers; the single legacy writer ceases when the last CRS booking/modify path is re-pointed; the table is retained read-only until Phase 11 (P-16).

**BR-5-019 — FO upgrade gate:** eligibility consults the authority; the legacy absent-row `true` default never gates a write.

**BR-5-020 — Error contract:** consumer-side detection of overbooking/assertion failures uses the deterministic error code contract (`AVAILABILITY_ASSERTION_REJECTED`, `rejection.code`), never message substrings (closes F-27's pattern).

**BR-5-025 — Single write truth (modify):** after ordering gates, reservation modify writes availability only through the assertion port; the `crs.modifyReservation` availability leg is removed.

**BR-5-026 — No premature cuts:** while the authority is not operational, write paths are unchanged (status quo preserved deliberately).

**BR-5-027 — Deterministic write identity (preserved):** explicit idempotency keys where available; never `hotelId || 'default'`; never absent identity minting random operation keys for availability-bearing writes (Phase 2 §5.4, extract `:177-185`).

**BR-5-050 — No legacy availability signal gates or publishes availability:** applies to `GET /rates/availability`, `GET /rates/engine/{availability,restrictions}`, `quote.available`, FO `isAvailable`, quick-book gate.

### 20.4 Restriction write ownership (DS-04)

**BR-5-021 — Authority-owned write path:** restriction writes are hotel-scoped, payload-validated, invalidate availability (TR-10.3 analogue for selling permission), and record provenance/audit; raw `$executeRawUnsafe` writes to restriction tables are prohibited after cutover (closes F-26 write side).

**BR-5-022 — Restriction reads for display** come from the authority projection; raw reads (`:384-405`) are removed.

**BR-5-023 — No second combination:** A3's availability overlay is removed (see BR-5-008).

**BR-5-024 — Unproven stores are not promoted:** `restrictions` (`rate_code='CUTOFF'`) and any store without evidenced meaning/population is not migrated into the authority write/read path (BR-5-046 + §30 E-4).

### 20.5 Scope, isolation, governance (DS-07, DS-08, DS-09, DS-10, DS-11, DS-12)

**BR-5-030 — Outbound publication:** no availability number may be published externally from a non-authority source; existing caller-supplied `channel_availability_log` writes are non-authoritative and feed nothing (DS-07 deferred).

**BR-5-031 / BR-5-032 — Hotel scoping:** every rule/read/write is hotel-scoped; no cross-hotel aggregation; `hotelId === 'default'` or missing property context is rejected at authority entry (P-15).

**BR-5-033 — Executed evidence per stage:** API full suite with `AVAILABILITY_TEST_DATABASE_URL` set (0 harness-skips for that reason), web typecheck/lint/test, baseline reconciliation reported in the stage artifact.

**BR-5-034 — Config declaration:** `AVAILABILITY_TEST_DATABASE_URL` declared in repo config documentation and CI (Stage 4 work, decided now).

**BR-5-035 — Single flag mechanism:** availability rollout flags = env `FEATURE_*` via `ConfigService.getFeatureFlag` only; no availability flag on the platform DB system; flag changes are operational, never a code side effect.

**BR-5-036 — Flag documentation:** all seven `FEATURE_*` flags declared in `compose.yaml`/`.env.example` (Stage 4 config work).

**BR-5-037 — `gba.a3.authoritative` ON gate:** evaluator shipped + U-3 population evidence + U-1 runtime confirmation + parity soak.

**BR-5-038 — Preserved rollout sequence:** `gba.consumers.cascade` ON at deploy; wash blocked by Deviations A+B; `canonicalRead` → soak → `canonicalWrite`; others default OFF.

**BR-5-039 — Freshness honesty:** the authority is `LIVE_READ`; no cache exists today; any future cache must implement TR-10.3 invalidation for real and may not be represented by a no-op delete.

**BR-5-040 — Canonical API names:** `/availability/logs` (GET+POST) and `/availability/interval-update` (POST, `ratePlans`/`dateRange{start,end}`) are the contract; frontend paths/payloads conform (`/availability/matrix/logs` and `/activities/…` are not contracts).

**BR-5-041 — No silent 404 affordance:** every availability UI control is either wired to its canonical contract or explicitly hidden with a declared status.

**BR-5-042 — Delete never releases:** hard-delete remains terminal-only (Decision 12) and performs no availability operation; balances and assertion audit rows are never released or cascade-deleted by deletion.

**BR-5-043 — Delete with availability state is rejected:** if `reservation_availability_state` / `reservation_availability_operations` rows exist for the reservation, delete is rejected with a deterministic business error before any mutation (schema `Restrict` intent, `schema.prisma:17399`, `:17426`); no raw FK error reaches the caller.

**BR-5-044 — One route, one owner:** duplicate route declarations are resolved by naming a canonical owner (after runtime verification) and removing the other; no aliases.

**BR-5-045 — `room_inventory` is legacy:** no availability figure sourced solely from `room_inventory` may be displayed or returned as availability.

**BR-5-046 — Input validity:** each authority input store has an identified writer/population source recorded; where none is found, the store's contribution is flagged with provenance (TR-10.4) or excluded — never silently trusted.

---

## 21. Blocker Classification Register

| ID | Class | Item | Reason | Affected consumer(s) | Governing rule(s) | Stage 3 requirement |
|---|---|---|---|---|---|---|
| **BLK-P5-01** | **HARD** | DS-01 evaluator + input population evidence + runtime confirmation | The binding is permanently `UNRESOLVED` ⇒ `sellableAvailable: 0` / `UNKNOWN` ⇒ assertion rejects (`AVAILABILITY_ASSERTION_REJECTED`) and flag-ON would zero the matrix | All authority consumers: assertion engine, `/snapshot`, A3 (flag ON), frontend cutover, booking re-point | BR-5-001…005, 009, 037, 046 | Requirement: "restriction evaluator over the evidenced store set with BR-5-002/003 semantics, proven by tests incl. conflict and absence cases, plus recorded U-1 runtime confirmation and U-3 population evidence" |
| **BLK-P5-02** | CONDITIONAL | DS-11 part 1 UI repair wired to restriction edits | Wiring before DS-04's authority write path would cement raw legacy writes | Availability page audit-log + interval-update UI | BR-5-021, 040, 041 | Requirement: "UI affordances implemented against the authority-owned restriction write contract; until then, affordances visibly disabled — never 404" |
| **BLK-P5-03** | CONDITIONAL | DS-03 booking-path retirement & DS-05 leg removal | Ordering requires the authority to be operational first, then readers disposed, then writers removed | Quick-book, FO upgrade, CRS book/modify/release, rates tab | BR-5-015…019, 025, 026 | Requirement: "retire/re-point in the order DS-01 → DS-03 reads → DS-05 write leg; no step skipped" |
| **BLK-P5-04** | NON-BLOCKING | DS-09 evidence policy | Gates stage exits, not decisions | All stages | BR-5-033, 034 | Requirement: "executed suite evidence recorded in each stage artifact" |
| **BLK-P5-05** | NON-BLOCKING | DS-10 flag documentation/governance | Rollout correctness, not rule correctness | Ops/release | BR-5-035…038 | Requirement: "flag inventory documented; no code-level flips" |
| **BLK-P5-06** | **DEFERRED** | DS-07 outbound push, DS-08 multi-property, DS-06 occupancy analytics contract | Ratified deferrals (P-19, P-20, TR-14.3) | Channels, corporate/multi-property, analytics | BR-5-030, 031, 028, 029 | Requirement: "explicitly out of scope; may not be partially implemented under another heading" |

---

## 22. Decision Dependency Map

Only evidence-backed edges (each cited):

```
DS-12 (E-1 population evidence, U-3)
   └──> DS-01.5 scope confirmation ──> BLK-P5-01
DS-01.2 evaluator (BR-5-001…004) ──> DS-01 operational
   ├──> DS-02 execution (cutover + BR-5-011 rendering of real states)
   ├──> DS-03 read retirement (BR-5-015/016)            [read side can start in parallel]
   ├──> DS-03 booking retirement (BR-5-017/019)         [needs operational authority]
   ├──> DS-05 write-leg removal (BR-5-025)              [needs DS-01 + DS-03]
   └──> DS-10 BR-5-037 flag-ON gate
DS-04 authority write path (BR-5-021) ──> DS-11 part 1 wiring (BR-5-040/041)
DS-09 (BR-5-033) ──> every later stage exit
DS-06 rules depend on DS-02 output contract (BR-5-028) for anything availability-labelled
Independent: DS-07, DS-08 (scope deferrals), DS-11 part 2 (delete), DS-11 part 3 (route owner)
```

**Hard ordering statement (carried to Stage 4):** *B-1 (authority enablement: DS-01 + DS-12 evidence) before B-2…B-6* — matching audit §14, now with decisions attached.

---

## 23. Traceability Matrix (F → DS → BR → consumer → Stage 3 requirement)

| Finding | DS | BR / status | Affected consumer | Stage 3 requirement |
|---|---|---|---|---|
| F-01 (permanently UNRESOLVED ⇒ assertion rejects) | DS-01 | BR-5-001…005, 009, 037 | Assertion engine, `/snapshot`, A3 (flag ON) | Evaluator requirement + conflict/absence cases + U-1/U-3 evidence |
| F-02 (0 consumers of authority read) | DS-02 | BR-5-010, 013 | Availability page, rates tab, quick-book, admin/mobile | `useAvailabilitySnapshot`/projection consumer requirements + consumption-count acceptance |
| F-03 (A3 second engine) | DS-04 | BR-5-008, 021, 022, 023 | A3 matrix + restriction endpoints | Authority-owned restriction read/write requirements; overlay removal |
| F-04 (dual-write drift) | DS-05 | BR-5-025, 026 | Reservation modify path, legacy `availability` | Ordered removal requirement + drift-monitoring via reconciliation |
| F-05 (FO upgrade legacy gate) | DS-03 | BR-5-019, 050 | Front Office upgrade | Eligibility consult via authority; absent-row default removed |
| F-06 (unscoped `/rates/availability`) | DS-03 | BR-5-015 | Rates-inventory page | Route retirement (or de-availability + hotel scoping) requirement |
| F-07 (legacy write gate) | DS-03 | BR-5-016, 017 | Quick-book, FO booking | Booking gate via authority eligibility; book path through reservation create |
| F-08 (404 logs + interval payload) | DS-11 | BR-5-040, 041 (+021 gating) | Availability page audit-log/interval UI | Canonical route/payload requirement; wiring gated by DS-04 |
| F-09 (client-side math ≥12) | DS-02 | BR-5-012, 028, 029 | All display surfaces (§5.3 list) | Remove/re-point each formula; conservation law for bed-type split |
| F-10 (quick-book always 0) | DS-02 | BR-5-010, 012 | Quick-book matrix | Cell mapping from projection + eligibility surfacing |
| F-11 (`room_inventory` no writer) | DS-12 | BR-5-045, 046 | `GET /availability/room-types` consumers | Availability figures re-pointed to authority; table treated as legacy |
| F-12 (contradictory numbers) | DS-02, DS-06 | BR-5-010, 028, 029 | Cross-screen workflows | Single-source acceptance: no two screens disagree on available |
| F-13 (no shared type) | DS-02 | BR-5-013 | All API consumers | Shared contract type requirement |
| F-14 (no-op invalidation) | DS-10 | BR-5-039 | Event consumers / freshness display | Document "no cache"; future cache must implement TR-10.3 |
| F-15 (no channel push) | DS-07 | BR-5-030 | Channels | Out of scope; non-authoritative log marked as such |
| F-16 (write identity gaps) | DS-05 | BR-5-027 | create/change-rate/webhook paths | Deterministic identity requirement (preserved Phase 2 §5.4) |
| F-17 (48 suites skip) | DS-09 | BR-5-033, 034 | CI / all stages | Env declaration + zero-harness-skip reporting |
| F-18 (dual flag systems, undeclared flags) | DS-10 | BR-5-035, 036 | Release/ops | Single mechanism + config documentation |
| F-19 (property-scope divergence) | DS-02 | BR-5-031, 032 (+047…049 isolation) | Authority controller, A3/Banquet controllers | Route-property authorization requirement; guard divergence reconciled (§25) |
| F-20 (`/tax-rates` collision) | DS-11 | BR-5-044 | Banquet/availability reference reads | Runtime winner verification + single owner |
| F-21 (dead code cluster) | DS-03/DS-04 | BR-5-018, 021 (+ Stage 4 hygiene) | Multiple | Removal gated behind behavior-preserving migration (B-8) |
| F-22 (baselines unreconciled) | DS-09 | BR-5-033 | Evidence quality | Baseline reconciliation in first executed stage |
| F-23 (Phase 4 doc drift) | DS-09 | BR-5-033 (evidence discipline) | Documentation | Correct drift in Stage 4 doc chores; never cite stale lines as live |
| F-24 (Phase 4 exit actions open) | DS-10 | BR-5-038 | Release/cutover | Phase 4 actions remain owned by Phase 4; Phase 5 may not assume them done |
| F-25 (duplicate web routes) | DS-02 | BR-5-010 (+ Stage 4 hygiene) | Nav/cache | Consolidation behind the sanctioned read surfaces |
| F-26 (raw-SQL interpolation) | DS-04 | BR-5-021 (write side); read side Stage 4 | A3 endpoints | Parameterized reads/writes on the authority path |
| F-27 (OTA overbooking branch) | DS-03 | BR-5-020 | OTA webhook handler | Deterministic error-contract detection; U-6 live check |

**Risk register dispositions (audit §11):** R-01→BLK-P5-01; R-02→BR-5-009/037; R-03→BR-5-005 + U-1; R-04→BR-5-017; R-05→BR-5-015; R-06→BR-5-025 (ordering); R-07→BR-5-021/022; R-08→BR-5-010/028; R-09→BR-5-040/041; R-10→BR-5-033; R-11→BR-5-036; R-13→TR-4.1 wiring remains a Phase 4 carry (`13_…:463`, DEF-3/T-41) — **not re-decided here**, tracked as an inherited dependency of BR-5-005 enforcement.

---

## 24. Findings Disposition (F-01…F-27)

| Finding | Disposition class |
|---|---|
| F-01, F-02 | **DECIDED-BY-RULE** (BLK-P5-01 open until implementation + evidence) |
| F-03, F-04, F-05, F-06, F-07, F-10, F-11, F-13, F-16, F-19 | **DECIDED-BY-RULE** (execution ordered per §22) |
| F-08, F-20 | **DECIDED-BY-RULE** + `UNRESOLVED` sub-item (U-2 runtime winner) |
| F-09, F-12, F-25 | **DECIDED-BY-RULE** (display/contract rules) with Stage 4 hygiene follow-through |
| F-14, F-23, F-24 | **DECIDED-BY-RULE** (governance/honesty) + Stage 4 chores; F-24 items remain Phase 4-owned |
| F-15, F-21 | **DEFERRED / Stage-4 hygiene** (gated: never precede behavior-preserving work) |
| F-17, F-22 | **DECIDED-BY-RULE** (DS-09 evidence policy) |
| F-18 | **DECIDED-BY-RULE** (DS-10 flag governance) |
| F-26 | **DECIDED-BY-RULE** (write side) + Stage 4 (read side) |
| F-27 | **DECIDED-BY-RULE** (error contract) + `UNRESOLVED` evidence (U-6) |
| Items verified compliant (audit §13.1–13.14) | **PRESERVED FROM PRIOR PHASE — DO NOT REOPEN** (incl. `CHECKED_OUT` non-release, D-0/D-1 consumption adapter, FO zero-GBA-writes, F-18 retirement, terminal-only delete, room-level FO scope, warehouse/banquet out of scope) |

---

## 25. Property / Tenant Isolation Rules

Binding for every Phase 5 rule above (P-15; INV-1 `13_…:593`; TR-14.1–14.5):

- **BR-5-047 — Route-property authority:** the property used for authorization must be the property used for data access. Where a route declares `:propertyId`, that value is authoritative and is validated against the principal's membership; header/tenant-derived scope may not silently select a different property (closes the F-19 divergence class).
- **BR-5-048 — No guard opt-out for availability surfaces:** `@PropertyScope(false)` controllers (`availability-sales.controller.ts:30-31`, `banquet-refs.controller.ts`) may not read or write availability-bearing data unless an equivalent property-scoped authorization is performed (the snapshot's `authorize()` pattern: membership + permission codes + `default` rejection, `availability-snapshot.service.ts:17-33`).
- **BR-5-049 — Raw SQL carries `hotel_id`:** every raw statement touched by Phase 5 work includes `hotel_id` in its predicate; bare-id mutation prohibited.
- **BR-5-051 — Unscoped reads are not preserved for compatibility:** any legacy read found unscoped (F-06) is retired or scoped regardless of consumer impact (§5 prohibition).
- Multi-property aggregation remains out of scope (BR-5-031).

---

## 26. Frontend Display and Eligibility-Surfacing Rules

(Consolidated view of BR-5-010/011/012/013/028/029/040/041 for Stage 3 consumption.)

1. Two sanctioned contracts only (BR-5-010); the availability grid uses the projection; anything needing provenance/eligibility uses the snapshot.
2. Three eligibility states must be visually distinguishable: `ELIGIBLE`, `UNKNOWN` (unresolved), `BLOCKED` (BR-5-011).
3. Unresolved zeros are never presented as certainty; interim label stays until authority-backed (BR-5-011, P-17).
4. No client-side availability/occupancy computation (BR-5-012); bed-type split conserves the authority figure exactly.
5. One shared payload type (BR-5-013).
6. Availability-labelled KPIs derive from authority; occupancy KPIs are declared reporting (BR-5-028/029).
7. No silent 404 controls; canonical routes/payloads only (BR-5-040/041).
8. Frontend may not hold snapshot data in legacy Zustand stores in a way that creates a second truth; React Query cache with explicit keys follows the existing convention (audit §5.4 confirmed no snapshot store exists today — preserved).

---

## 27. API Contract Rules for Phase 5

1. **Authority surface is fixed:** `GET /properties/:propertyId/availability/snapshot` and `.../reconciliation` (P-1); no new availability endpoints may be invented for convenience (plan §8 discipline, `14_…:813-840`).
2. **Matrix is a projection:** shape preserved; additive eligibility fields permitted; values from authority when flag ON; label when OFF (BR-5-010, P-18).
3. **Canonical restriction UI contract:** `/availability/logs` (GET/POST), `/availability/interval-update` (POST, `ratePlans`/`dateRange{start,end}`), `/availability/bulk-update` (already matching) — BR-5-040.
4. **Retirements:** `GET /rates/availability`, `/rates/engine/{availability,restrictions,book,release}` per §10.2 (404 acceptable — contract requires retirement, not compat alias; precedent T-39).
5. **Error contract:** deterministic codes (`AVAILABILITY_ASSERTION_REJECTED`, `rejection.code ∈ {UNRESOLVED_CAPACITY, CAPACITY_BLOCKED, INSUFFICIENT_CAPACITY, …}`); consumers must match codes, not strings (BR-5-020).
6. **DTO validation:** authority and A3 handlers gain typed DTOs for availability-bearing inputs (audit §7.3 gap) — Stage 3 requirement, decided here as a duty: *no availability endpoint accepts unvalidated `body: any` for writes*.
7. **Isolation:** §25 rules apply to every contract above.
8. **Gateway:** no availability-specific routing/versioning exists (audit §7.5) — none is invented; the catch-all stays (preserved).

---

## 28. Phase 5 Scope Boundary

**In scope (decided here):** authority enablement requirements (DS-01/DS-12), frontend read-contract migration rules (DS-02), legacy consumer dispositions (DS-03), restriction ownership (DS-04), write convergence (DS-05), display/KPI rules (DS-06), UI repair rules (DS-11), test/flag governance (DS-09/DS-10).

**Out of scope (explicitly, with status):**

| Item | Status | Basis |
|---|---|---|
| Outbound channel/CRS push contract (B-6) | `DEFERRED` | P-19 (`13_…:65`) |
| Multi-property contracts (B-7 corporate) | `DEFERRED` | TR-14.3 (P-15) |
| Occupancy analytics contract (formula, realtime vs batch) | `DEFERRED` | P-20 (`13_…:67`) |
| Schema changes, migrations, data backfills | prohibited by user constraint | AGENTS.md; Phase 4 deviation C |
| Legacy table deletion | Phase 11 | P-16 (`13_…:731-735`) |
| Phase 4 open actions (commit delta, canonical cutover, cascade flag, BLK-1/2) | Phase 4-owned | `17_…:92-99` (F-24) |
| Pickup ↔ assertion port shape (DEF-3/T-41) | Inherited, not re-decided | `16_RR_DECISIONS.md:28-38` |
| Hygiene: dead code (F-21), duplicate routes (F-25), doc drift (F-23) | Stage 4, gated **after** behavior-preserving work | audit §14 B-8 |

**Migration-boundary constraint (carried):** B-1 (DS-01 + DS-12 evidence) must precede B-2…B-6; B-4 (write convergence) is the highest-risk boundary and follows B-1 and the DS-03 dispositions.

---

## 29. Phase 4 Carry-Overs and Non-Reopened Decisions

| Carry-over | State preserved | Phase 5 dependency |
|---|---|---|
| Deviation A (BLK-1: `GUARANTEED_BLOCK` wash exclusion not shipped; t45 test skipped) | Preserved — hard precondition before wash activation | None for Phase 5 decisions; wash UI labels stay governed by `gba.wash.schedulerEnabled` (BR-5-038) |
| Deviation B (BLK-2: durable wash/release/attrition store undecided) | Preserved | Same as A |
| Deviation C (Phase 2 DB foundation skipped) | Preserved — no schema changes | Reinforces §30 guardrail G-1 |
| Deviation D (NB-1…NB-6 doc chores) | Preserved | Absorbed into Stage 4 doc tasks (F-23) |
| DEF-1…DEF-10 (mechanisms) | Preserved as mechanisms, never as business rules (`14_…:981`) | Not re-opened |
| Flags & rollout sequence | Preserved (P-21) | BR-5-035…038 |
| Test baselines | Preserved (P-22) | BR-5-033 reconciliation |
| A3 interim label, matrix shape preservation | Preserved (P-17, P-18) | BR-5-010/011 |
| `CHECKED_OUT` non-release; terminal-only delete; consumption adapter semantics | Preserved (audit §13.1/13.2/13.8) | BR-5-042/043 clarification only |

---

## 30. Evidence Requirements and Stage-3 Guardrails

### 30.1 Evidence requirements (named, never defaulted)

| ID | Evidence required | Closes | Gates |
|---|---|---|---|
| **E-1 (U-3)** | Population audit: per-hotel row counts + recency + writer identification for `room_inventory`, `out_of_order`, `out_of_service`, `rate_restrictions`, and the six A3-written restriction tables | DS-12, DS-01.5 | BLK-P5-01 |
| **E-2** | Precedence-intent evidence: product/operational documentation of how stores should rank when they disagree (only needed if actual conflicts exist) | DS-01.3 explicit ranking (currently conflict ⇒ UNRESOLVED) | Nothing (fail-closed default stands) |
| **E-3 (U-1)** | Executed confirmation that the production DI binding rejects/asserts as the static chain predicts (e.g. a create attempt observed end-to-end) | DS-01.6 | BLK-P5-01, BR-5-037 |
| **E-4** | Business meaning of `restrictions` (`rate_code='CUTOFF'`) and the `zero_sell_value` domain | DS-04 sub-item, BR-5-024 | Evaluator scope for that store only |
| **E-5 (U-5)** | Inventory of out-of-repo consumers of `/rates/engine/*` | DS-03 schedule | Retirement timing only |
| **E-6 (U-6)** | Live behavior of the OTA overbooking branch | DS-03 (BR-5-020) | None (rule already issued) |
| **E-7 (U-2)** | Runtime winner of the `/tax-rates` collision | DS-11 part 3 | BR-5-044 execution |
| **E-8 (U-4)** | Executed baseline reconciliation | DS-09 | First Stage 3+ exit |

### 30.2 Stage 3 guardrails (must not be done, under any convenience argument)

- **G-1** No schema, migration, index, constraint, or seed changes (user constraint; Phase 4 deviation C).
- **G-2** No fail-open: never suppress `unresolvedSources`, never treat `UNKNOWN` as `ELIGIBLE`, never bypass `unassertableReason` (P-3/P-5/P-6/P-7).
- **G-3** No invented restriction policy: no precedence ranking, no invented closure/LOS/sell-limit semantics (BR-5-003; §30 E-2 is the only path to a ranking).
- **G-4** No legacy-comes-first compatibility: retiring or scoping a legacy surface may not be vetoed by its current consumers (BR-5-051, §5).
- **G-5** No flag flips inside code changes; no new availability flags on the platform DB system (BR-5-035).
- **G-6** No test-execution claims without recorded runs (BR-5-033); no reliance on harness-skipped suites as evidence.
- **G-7** No cross-hotel aggregation, no `hotelId==='default'` fallback, no unscoped raw SQL (BR-5-031/032/049).
- **G-8** No third availability contract, no client-side availability math, no second combination site (BR-5-008/010/012).
- **G-9** No partial scope creep into deferred items (DS-07/DS-08/occupancy contract) under another heading (BLK-P5-06).
- **G-10** No deletion of availability audit/balance rows from a reservations path (BR-5-042/043).
- **G-11** Ratified Phase 1–4 decisions (§4 register) may not be re-opened; changes require the §5 amendment protocol with new evidence.
- **G-12** Hygiene/dead-code work may not precede behavior-preserving migration (audit §14 B-8).

---

## 31. Decision Closure Summary

### 31.1 Decision items

| DS | Question (short) | Status | Blocker class | Rules | Stage 3 entry condition |
|---|---|---|---|---|---|
| **DS-01** | Restriction authority / can the authority assert? | `NEW DECISION` (Option A) with sub-items: DS-01.1/01.7 `PRESERVED`, DS-01.3/01.4 `NEW DECISION`, **DS-01.5/01.6 `UNRESOLVED — EVIDENCE REQUIRED`** | **HARD** (BLK-P5-01) until E-1 + E-3 + implementation | BR-5-001…009, 037, 046 | Evaluator requirement drafted; E-1 and E-3 scheduled |
| **DS-02** | Frontend read contract | `NEW DECISION` (Option C: snapshot canonical + matrix projection) | Released as a decision; execution gated by BLK-P5-01 | BR-5-010…014 | Contract types + eligibility-surfacing requirements specified |
| **DS-03** | Legacy rates/CRS disposition | `NEW DECISION` (retire/re-point per §10.2) | `CONDITIONAL` (BLK-P5-03) | BR-5-015…020, 050 | E-5 external-consumer inventory before retirement schedule |
| **DS-04** | Restriction write ownership | `NEW DECISION` (authority-owned path) + sub-item `UNRESOLVED` (CUTOFF meaning) | `CONDITIONAL` (BLK-P5-02) | BR-5-021…024 | Write-path requirement; E-4 before any CUTOFF migration |
| **DS-05** | Dual-write ordering (+ write identity) | `NEW DECISION` (ordered removal; status quo preserved until gates) | `CONDITIONAL` (BLK-P5-03) | BR-5-025…027 | Ordering stated in requirements verbatim |
| **DS-06** | Occupancy/KPI | availability figures `RESOLVED`; occupancy analytics contract `PRESERVED` as `DEFERRED` | `NON-BLOCKING` / `DEFERRED` | BR-5-028, 029 | Display rules specified; analytics contract not started |
| **DS-07** | Channel push scope | `PRESERVED FROM PRIOR PHASE` → `DEFERRED` | `DEFERRED` | BR-5-030 | Out of scope declaration carried |
| **DS-08** | Multi-property scope | `PRESERVED FROM PRIOR PHASE` → `DEFERRED` | `DEFERRED` | BR-5-031, 032 | Out of scope declaration carried |
| **DS-09** | Test-execution policy | `NEW DECISION` | `NON-BLOCKING` (gates stage exits) | BR-5-033, 034 | E-8 baseline reconciliation in first executed stage |
| **DS-10** | Flag governance | `NEW DECISION` + rollout sequence `PRESERVED` | `NON-BLOCKING` | BR-5-035…039 | Config documentation task; no code flips |
| **DS-11** | UI repair / route collision / delete | part 1 `NEW DECISION`; part 2 `PRESERVED` + `NEW DECISION`; part 3 `UNRESOLVED` (U-2) | part 1 `CONDITIONAL`; parts 2–3 `NON-BLOCKING` | BR-5-040…044 | Canonical contracts fixed; E-7 before route-owner change |
| **DS-12** | Physical/OOO/restriction inputs | ownership `RESOLVED`/`PRESERVED`; **population `UNRESOLVED — EVIDENCE REQUIRED`** | **HARD** with BLK-P5-01 (evidence part) | BR-5-045, 046 | E-1 population audit |

### 31.2 Counts

| Metric | Value |
|---|---|
| Decision items evaluated | 12 (DS-01…DS-12) |
| Sub-decisions additionally recorded | DS-01 ×7, DS-04 ×1, DS-11 ×3, DS-12 input table (11 stores) |
| Business rules issued | 51 (BR-5-001…BR-5-051) |
| Preserved prior-phase decisions cited | 22 (P-1…P-22) |
| Findings dispositioned | 27 / 27 (F-01…F-27) |
| Hard blockers | 1 (BLK-P5-01) |
| Conditional blockers | 2 (BLK-P5-02, BLK-P5-03) |
| Non-blocking / deferred classes | 3 + 3 (BLK-P5-04/05/06, DS-07/08/occupancy deferrals) |
| `UNRESOLVED — EVIDENCE REQUIRED` items | 6 (E-1…E-4, E-7, E-8) + 2 declared evidence-only (E-5, E-6) |
| Code/schema/test/config changes made | **0** |

### 31.3 Closure statement

All twelve Stage 1 decision items have been evaluated against the ratified Phase 1–4 corpus and repository evidence. Every item now carries a status, rules, a blocker class, and a Stage 3 requirement; every finding F-01…F-27 is dispositioned in §23/§24; nothing was silently dropped, and no restriction policy was invented — where evidence was insufficient the item is declared `UNRESOLVED` with the exact missing evidence (§30.1).

**The single hard blocker remains DS-01**, now decomposed into: a decided implementation direction (Option A), two preserved fail-closed semantics, two evidence-derived handling rules (absence, conflict), and two evidence gates (E-1 population audit, E-3 runtime confirmation). No downstream migration (DS-02 execution, DS-03 booking retirement, DS-05 write-leg removal, `gba.a3.authoritative`) may begin before BLK-P5-01 closes.

**Next step:** Stage 3 (Requirements) may begin, producing the Stage 3 requirements referenced in §23/§31.1 — still without implementation. Implementation, migrations, test execution, route/flag changes remain prohibited until the corresponding later stages and §30.2 guardrails are satisfied.

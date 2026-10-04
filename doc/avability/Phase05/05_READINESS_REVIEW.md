# Phase 5 — Stage 5: Readiness Review

## 1. Document Control

| Field | Value |
|---|---|
| Artifact | `docs/availability/phase-5/05_READINESS_REVIEW.md` |
| Stage | 5 of 6 (Readiness Review) — STRICT READ-ONLY |
| Date | 2026-10-04 |
| Authority | Local workspace `C:\Users\Pro\Desktop\XYLO` only (GitHub out of scope) |
| Change boundary | This file is the **only** artifact created/modified; zero source/schema/test/config/flag/DB changes |
| Git dirty count before review | **776** (all pre-existing, `phase-5/` shown as one untracked dir) |
| Verdict | **READY WITH NON-BLOCKING NOTES** (see §33) |

## 2. Review Scope

Determine whether the approved Phase 5 Implementation Plan (`04_IMPLEMENTATION_PLAN.md`) is genuinely ready for Stage 6 implementation, using the §5 readiness question:

> Can a competent implementation agent execute T5-01 through T5-90 exactly as planned without making undocumented business, architecture, API, data, security, property-isolation, or migration decisions?

**In scope:** plan integrity, Stage 3→4→5 traceability, blocker/evidence readiness, DS-01 readiness, DB/isolation/API/consumer/frontend/legacy/cutover/flag/log/test/scan/rollback/dependency reviews, gate R1–R12, findings, Stage 6 entry conditions.

**Out of scope:** implementation (T5-01…T5-90), Stage 6 start, any test execution beyond lightweight read-only file/count verification, any plan rewrite (defects are reported, never silently edited).

## 3. Authority Hierarchy

1. Approved Phase 1–4 domain/decision authority (`docs/availability/phase-4/*`, ADRs)
2. Phase 5 `02_BUSINESS_RULES_DECISIONS.md` (BR-5-001…051)
3. Phase 5 `03_FINAL_DOMAIN_SPECIFICATION.md` (S3R registry, AC/INV/M, DS-01…12)
4. Phase 5 `04_IMPLEMENTATION_PLAN.md` (WS/tasks/gates)
5. Local source/schema/tests evidence (read-only inspection)

No GitHub/remote content consulted or required.

## 4. Input Artifacts

| Artifact | State | Read |
|---|---|---|
| `01_FORENSIC_AUDIT.md` | CLOSED | §5–§7, §10, §14, §15 inventories (re-verified against source) |
| `02_BUSINESS_RULES_DECISIONS.md` | CLOSED | §20 catalogue (51/51 unique BR IDs re-counted), §21–§24, §30 |
| `03_FINAL_DOMAIN_SPECIFICATION.md` | CLOSED | §7.3 (S3R classes), §13/§16/§21/§24/§30–§36 |
| `04_IMPLEMENTATION_PLAN.md` | CLOSED | all 37 sections, 90 task blocks, chains, registers (structurally re-verified) |
| Phase 4 `17_PHASE4_READINESS_REVIEW.md`, `14_IMPLEMENTATION_PLAN.md` | referenced | Deviations A–D, BLK-1/2, baselines (via FDS §32 + plan §6/§32) |
| Local source (read-only) | inspected | ~25 anchor checks — results inline in §13–§24 |

## 5. Stage 4 Validation

| Check | Result | Evidence |
|---|---|---|
| `04` exists, 37/37 sections, no continuation marker | PASS | section scan `## 1.`…`## 37.`; 0 `CONTINUE` |
| 14 workstreams WS-A…WS-N declared | PASS | plan §11 table (A–N, task ranges sum: 4+7+7+10+10+6+4+5+3+5+2+5+8+10 = 90) |
| 90 unique task IDs T5-01…T5-90, each defined once | PASS | 81 heading defs + 10 table defs = 90 unique, no duplicates, none out of range (T5-19 bold cross-ref note at line 431 excluded from count) |
| 7 dependency chains CHAIN-1…7 | PASS | plan §8.2 |
| P0–P6 phases + G1 serial gate | PASS | plan §29 |
| Rollback conditions present | PASS | plan §28 table (14 switches) + per-task Risk/Rollback |
| Evidence gates E-1…E-8 registered | PASS | plan §10 |
| Stage 4 changed nothing outside `04` | PASS | git dirty 776 before Stage 4 → 776 after; `phase-5/` contains only `01`–`04` |
| Baselines preserved | PASS | plan §6: API 178/1470/1skip/6fail; GBA 79/766/1skip/0fail; web 1-suite/10-test; 51+2 migrations |

## 6. Artifact Completeness Confirmation

- Phase-4 parity verdict STATE B (plan §4) re-accepted: all 20 Phase-4 layers mapped to `01/02/03` with citations; the single declared data gap (population facts) is evidence gate E-1, not a missing artifact. No State C condition exists.
- Stage 3→4 handoff inputs (FDS §36) all present in `04`: baselines (§6), tasks (§12–26), evidence scheduling (§10), scope fence (§36), doc-consistency duties (§5/§63).
- Stage 5 handoff outputs required by `04` §35 are deliverable by this review: evidence status known, gate tasks defined, baselines frozen, G-1 confirmed.

## 7. Stage 3 Traceability Validation

Re-counted programmatically this session:

| Dimension | Expected | Found | Coverage |
|---|---|---|---|
| Business rules BR-5-001…051 | 51 | 51 unique IDs in `02`; 51 rows mapped in `04` §31.1 | 51/51 |
| Acceptance AC-01…47 | 47 | 47 defined in FDS §30; 47 rows in `04` §27.1 | 47/47 |
| Invariants INV-P5-01…34 | 34 | 34 unique in FDS; 34 rows in `04` §27.2 | 34/34 |
| Migration rules M-1…M-10 | 10 | 10 in FDS (M-21 hit = audit finding label, false positive); 10 rows in `04` §27.3 | 10/10 |
| S3R-001…116 | 116 | FDS §7.3 class tables (ranges: A=001–051, B=052–057, C=058–065, D=066–077, E=078–089, F=090–116) all mapped in `04` §31 | 116/116 |
| Cross-cutting groups S3R-G1…G6 | 6 | FDS §7.3 + `04` §31.7 | 6/6 |
| Orphaned requirements / dropped items | 0 | FDS §13.7: "Silently dropped: 0"; no task lacks a Why/Authority cite | 0 |
| Orphaned tasks (no S3R/AC anchor) | 0 | every `04` task carries Why/Authority citing BR/AC/INV/M/S3R/F/DS | 0 |

**Result: complete bidirectional traceability Stage 3 → plan → verification → gate.**

## 8. Domain Integrity Review

| Check | Result | Evidence |
|---|---|---|
| No Stage 2 decision reopened | PASS | `04` §3 principle + §5 status tallies match FDS (88 SPECIFIED / 10 PRESERVED / 16 BLOCKED / 2 DEFERRED) |
| No Stage 3 rule redesigned | PASS | task authorities cite FDS sections; T5-01/T5-08 semantics restate BR-5-001…006 verbatim (floors, conflict→UNRESOLVED, absence→RESOLVED+provenance, outcome integrity) |
| Fail-closed / UNKNOWN / sellableAvailable / bookingEligibility semantics intact | PASS | `04` T5-01 Action mirrors source (`availability-snapshot.service.ts:104-107` observed: `bookingEligibility = …'UNKNOWN'/'BLOCKED'/'ELIGIBLE'`, `freshness LIVE_READ` verified live) |
| Interim rules (flag OFF, no write-gating re-point, stub retained) intact | PASS | `04` T5-11/T5-29/T5-33/T5-55 + §8.3 |
| Deferral boundaries intact (analytics contract, push, multi-property, wash) | PASS | `04` §33 matches FDS §34 item-for-item |
| No new business rule introduced by the plan | PASS | spot-audit: every Action traces to an existing BR/DS/AC; where two compliant implementations exist the constraint is stated (T5-22 disposition sanctioned by FDS §21.4 alternatives) |

**Gate R1: PASS.**

## 9. Requirement Integrity Review

| Item | Status | Verified where |
|---|---|---|
| BR-5-001…051 → spec → task → test → gate | 51/51 | `04` §31.1 (per-rule WS/task/verification/gate columns; 4 E-1/E-4/E-7-blocked rules flagged, matching FDS §7.3.1) |
| AC-01…47 → task → verification | 47/47 | `04` §27.1 |
| INV-P5-01…34 → task → verification | 34/34 | `04` §27.2 |
| M-1…M-10 → task → cutover verification | 10/10 | `04` §27.3 |
| S3R → WS → task → file → test → gate | 116+6 | `04` §31.1–31.7 |
| Critical behaviors have a validation path | PASS | e.g. AC-07/08/09/10←T5-08; AC-13/14←T5-19/64; AC-25←T5-24; AC-34←T5-59/60; AC-37/38/39←T5-54/55/56; AC-44←T5-62 |

Field-format deviation (classified in §29): 64 tasks carry the full 9-field block; 26 tasks (T5-39…44, T5-69…76, T5-77…86, T5-87…90) use compact or tabular form carrying equivalent substance (authority, action, verification, failure/rollback, completion) but not all nine literal labels; plan §11's blanket wording slightly overstates this. **No missing decision** — see NB-3.

**Gate R2: PASS.**

## 10. Blocker Review

| Blocker | Status | Reason | Evidence required | Tasks affected | Blocks Stage 6? | Independent work? | Unblock condition | Verification |
|---|---|---|---|---|---|---|---|---|
| **BLK-P5-01** | OPEN — HARD (conditional) | Evaluator unshipped; E-1/E-3 unknown | E-1 record (T5-05), E-3 runtime record (T5-06), evaluator+suite green (T5-07/08), binding swapped (T5-09), parity soak (T5-10) | T5-07/09/10/13/14/24/30/31/32(partial)/35/40/50/56 | **No** — only the gated set | Yes: P0 tasks (evidence, pins, API hardening, isolation, scans, flags, logs, WS-F guards, tests) all ungated | all five records green | recorded evidence + suite green (`04` §9) |
| **BLK-P5-02** | OPEN — CONDITIONAL | Restriction-edit UI would cement raw writes | T5-45/46 landed; affordance wiring or visible-disable state | T5-48, T5-33 (affordance portion) | No | Yes (interim disabled state defined) | write path + handlers-as-callers landed | AC-45 test |
| **BLK-P5-03** | OPEN — CONDITIONAL (ordering) | DS-01→DS-03→DS-05 order | per-step T5-53 checklist | T5-24, T5-50, retirement timing T5-22…25 | No | Yes (P0–P3 proceed) | CHAIN-1 closed + readers disposed | T5-53 checklist per step |
| **BLK-P5-04** | OPEN — NON-BLOCKING | Executed suite evidence per exit | exit battery reports | T5-76 + every exit | No | Yes | each exit records evidence | exit gate report |
| **BLK-P5-05** | OPEN — NON-BLOCKING | Flag governance | docs + zero code flips | T5-54/55/56 | No | Yes | flags documented; no flips | T5-55 + doc diff |
| **BLK-P5-06** | DEFERRED w/ authority | Out-of-scope (P-19/P-20/TR-14.3) | n/a — **no tasks** (G-9) | none | No | n/a | future explicit authority | `04` §36 non-inputs |

No blocker closed, downgraded, or converted into an assumption during Stage 5. BLK-P5-01 is a **conditional blocker**: it blocks exactly the tasks the plan gates, and the plan correctly prevents those tasks from starting on assumption (`T5-07 [BLOCKED: E-1 scope]`; `T5-09` requires all four records; §8.3 normative rules 1–5).

**Gate R4: PASS.**

## 11. Evidence Gate Review

| Gate | Required evidence | Available now? | Source | Validation | Dependent tasks | Status | Consequence if absent |
|---|---|---|---|---|---|---|---|
| **E-1** | per-hotel counts/recency/writers for `room_inventory`, `out_of_order`, `out_of_service`, `rate_restrictions` + 6 A3 restriction stores | **No** (task T5-05 exists, evidence not yet recorded) | read-only DB queries + repo writer search | evidence checklist (`04` §10) | T5-07 scope, T5-09, chain closure | OPEN — hard | store excluded / flagged per BR-5-046; BLK-P5-01 stays open; **no assumptions** |
| **E-2** | precedence intent (only if real conflicts) | deferred | product docs | conflict found → STOP | none while default holds | **DEFERRED (default: conflict → UNRESOLVED)** — status preserved exactly | fail-closed default stands; no ranking invented |
| **E-3** | runtime confirmation of stub→UNRESOLVED→409 chain | **No** (T5-06) | safe-environment execution | request/response/persisted-state record | T5-09, T5-56 | OPEN — hard | BLK-P5-01 open; flag stays OFF |
| **E-4** | CUTOFF meaning | No (T5-87) | product/domain | documented definition or "unknown" | T5-47 scope only | OPEN — non-blocking | store stays excluded (AC-31) |
| **E-5** | external consumer inventory of `/rates/engine/*` | No (T5-88) | ops/gateway review (local only) | inventory recorded | timing only of T5-22…25 | OPEN — non-blocking for disposition | in-repo retirements proceed; external timing waits |
| **E-6** | OTA branch live behavior | No (T5-89) | sandbox observation | observation recorded | none (rule stands) | OPEN — informative | BR-5-020 unchanged |
| **E-7** | `/tax-rates` runtime winner | No (T5-90) | runtime trace | responder observed | T5-27 | OPEN — non-blocking | route untouched, current behavior preserved |
| **E-8** | reconciled executed baselines | No (first executed exit) | full runs w/ DB env | numbers vs §6 baselines explained | every code-changing exit | OPEN — **first executed exit** | exit blocked; baselines never redefined |

**E-1/E-3 gating correctness confirmed:** T5-07's Preconditions = "E-1 recorded fixes the store set; E-4 pending = store stays excluded"; T5-09 = "T5-05, T5-06, T5-07, T5-08 all green"; §8.3 rule 1 forbids CHAIN-2/3/4 before CHAIN-1 evidence. No evidence fabricated; all evidence tasks are read-only and explicitly non-coding (`04` §10 preamble).

**E-8 role confirmed:** T5-76 = first executed exit, entry criterion for Stage 5→6 continuation (`04` §25, §35), reconciliation table filed with deviations explained (Stage 6 exit battery item 6).

**Gate R3: PASS.**

## 12. DS-01 Readiness

| Element | Specified where | Task | Gate |
|---|---|---|---|
| Evaluator authority + requirement text | FDS §13.2–13.5 (restated in `04` T5-07 Action verbatim) | T5-07 | **E-1 before scope** |
| Current unresolved stub understood | `availability.module.ts:26` → `UnresolvedRestrictionAdapter` **verified live this session**; stub kept as rollback artifact (FDS §13.7) | T5-09 (keeps stub), T5-11 (pins) | — |
| Required evidence explicit | E-1 (store set) + E-3 (runtime chain) — see §11 | T5-05, T5-06 | hard |
| room_inventory evidence | E-1 row + **excluded from evaluator** (BR-5-045) | T5-05 lists it for population facts; T5-47 asserts exclusion | — |
| OOO/OOS evidence | E-1 row (`out_of_order`, `out_of_service`) | T5-05 | — |
| rate_restrictions evidence | E-1 row (writer unknown → identified by T5-05) | T5-05 | — |
| A3 restriction population evidence | E-1 six stores (`close_to_arrival`, `close_to_departure`, `zero_sell_limits`, `room_type_sell_limits`, `minLos`, `maxLos`) — T5-05 scope lists **10 stores + 2 excluded** | T5-05 | — |
| Fail-closed behavior | BR-5-005; `04` T5-01 floors + T5-07 read-failure→UNRESOLVED | T5-01/T5-07/T5-08 | — |
| UNKNOWN semantics | BR-5-001/004; source behavior verified (`…:105` eligibility ternary) | T5-01/T5-03 | — |
| `sellableAvailable` semantics | FDS §11 floors; `04` T5-01 `min(capacityWithOverbooking, max(0, sellLimit))` pin | T5-01 | — |
| `bookingEligibility` semantics | FDS §15 state space A–G; distinctness pins | T5-03 | — |
| Assertion interaction | FDS §14; rejection codes `UNRESOLVED_CAPACITY`/`CAPACITY_BLOCKED`/`INSUFFICIENT_CAPACITY` (BR-5-005, AC-12) | T5-04/T5-26 | — |
| Conflict/absence cases (BLK-P5-01 proof) | BR-5-002/003 restated in T5-07 + mandated suite | T5-08 | closure evidence |
| Implementation task gate | `T5-07 [BLOCKED: E-1 scope]`, `T5-09 [GATE: E-1+E-3+T5-07/08]` | — | correct |

**No implementation behavior depends on an unresolved business decision:** store membership comes from recorded evidence (or exclusion), not invention; conflict ranking is prohibited, not open; CUTOFF/channel_restrictions excluded by rule. Everything else is restated requirement text.

**DS-01 portion: READY** (gated correctly; not blocked from Stage 6 start — only from its own gated tasks).

## 13. Database / Schema Readiness

**Position: ZERO DB / ZERO SCHEMA CHANGE — verified.**

| Check | Result | Evidence |
|---|---|---|
| Task requiring schema migration | **None found** | scan of all 90 tasks: no `db:generate`, `db:migrate`, `db:push`, DDL, or `schema.prisma` edit task; only T5-62 (gate, read-only) and T5-63 (citation currency) touch the area |
| Hidden backfill/data mutation | **None** | T5-05/T5-06 explicitly read-only; T5-45 writes rows only via **existing** tables (provenance reuses `availability_matrix_logs` — `POST /availability/logs` observed writing it with `INSERT … hotel_id` at `availability-sales.controller.ts:760-70`); assertion writes use existing Phase 2 tables |
| Assumes nonexistent columns/tables | **No** | T5-59 relies on `reservation_availability_state`/`reservation_availability_operations` (schema `:17399`/`:17426` verified: relation + `onDelete` Restrict intent); evaluator reads existing restriction stores |
| Hidden data mutation | **No** | delete-reject is a no-op-on-reject (AC-34 pin); reconciliation read-only (T5-57); no task mutates rows outside sanctioned write paths |
| DB evidence requirements explicit | Yes | E-1 (read-only queries), E-8 (runs with `AVAILABILITY_TEST_DATABASE_URL`), T5-70 (48 postgres suites, 0 env-skips) |
| Property-scoped queries safe | Yes — see §14 | T5-65/79/86 scans; T5-45 parameterized+scoped by design |
| Baseline migration count guard | Yes | T5-62 asserts 51 finished + 2 historical unfinished unchanged; `prisma validate` ×2 |

Baseline re-verified read-only: **178** `*.spec.ts`, **48** `*.postgres.spec.ts` in `apps/api/src` (matches plan §6).

Any task discovered to need schema work during Stage 6 would be a deviation requiring escalation — **not** silent schema change (plan §23 T5-62 fails the exit).

**Gate R8 (DB half): PASS.**

## 14. Property Isolation Readiness

| Path | Plan treatment | Verified evidence |
|---|---|---|
| Authority API | T5-64: 403 without context; `'default'` rejected; non-member 403; route-vs-header reconciliation (route wins) | `availability-snapshot.service.ts:17-33` authorize() **verified live**: rejects missing/`default` actor-hotel, checks membership + permission codes (`availability:read…`), returns frozen `AuthorizedPropertyContext` |
| Scoped reads | T5-65 scan + T5-19 e2e | A3 raw restriction reads **verified hotel-scoped** (`WHERE hotel_id = $1` at `availability-sales.controller.ts:384-405` block) |
| Scoped writes | T5-45 (parameterized, `hotel_id` predicate), T5-65 | `POST /availability/logs` verified: `INSERT INTO availability_matrix_logs (hotel_id, …)` |
| **Previously unscoped availability query** | **Remediation identified**: `rates-inventory.service.ts` `getAvailability()` — `prisma.rooms.count()` + `prisma.reservations.count()` with **no hotel filter** verified live at `:34-40` | T5-22 (retire/de-availability, scoped-retained-count option) + T5-77 scan names this exact file:lines + T5-51 scoping duty; BR-5-051/015 |
| Jobs/workers | T5-67 (`events.consumer.ts:140-202` hotel-prefixed invalidation; no cross-hotel aggregation) | audit-backed |
| Logs | T5-64 scoping suite + `availability_logs` reads scoped | plan §24 |
| Reconciliation | T5-57 (read-only, hotel-scoped) | plan §21 |
| API calls | T5-19 (route property governs) | plan §16 |
| Frontend requests | T5-68 + T5-29 queryKey includes propertyId | plan §24 |
| No global fallback introduced | T5-86 scan (`hotelId \|\| 'default'`, `'default'` literals, bare-id mutations) | plan §26 |
| Guard infrastructure exists | `property-scope.guard.ts`, `partition-router.middleware.ts` located in repo | glob re-verified |

**Gate R8 (isolation half): PASS.**

## 15. API Readiness

| Endpoint | Current contract (verified) | Target contract | Consumers | Order | Compat/retirement | Tests |
|---|---|---|---|---|---|---|
| `GET /availability/snapshot` | exists (`availability.controller.ts:12`), authorize pattern live | unchanged (pinned) | 0 today (audit §15.3-E1) | ungated | additive types only (T5-02) | T5-19, T5-01 |
| `GET /availability/reconciliation` | exists (`:20`) | read-only, unchanged | backend readers | ungated | flagged deltas only | T5-19, T5-57 |
| `GET /availability/matrix` | exists (`availability-sales.controller.ts:265`), flag `:275`, label `:612`, overlay `:540` — **all verified live** | shape never changes; OFF=label+interim values; ON=authority facts | quick-book, AvailabilityPage, FO | after G1 for values | overlay removed only in ON branch (T5-40) | T5-20, T5-10 |
| `GET/POST /availability/logs` | exist (`:719/:752`) writing `availability_matrix_logs` | canonical (frontend conforms) | frontend logs view | ungated | replaces broken `/availability/matrix/logs` caller (`reservation.api.ts:405/420` verified) | T5-33, T5-21 |
| `POST /availability/interval-update` | exists (`:773`) expects `ratePlans`+`dateRange{start,end}` | canonical payload | `reservation.api.ts:431` currently sends `ratePlanCodes/startDate/endDate` to `/activities/availability/interval-update` (**404 verified as broken caller**) | ungated conformance | fixes silent 404 | T5-33, T5-21 |
| `GET /rates/availability` | exists (`rates-inventory.controller.ts:34`), unscoped computation | RETIRE (or de-availability; default=retire per FDS §21.4) | `rates-inventory/page.tsx:143,155` | E-5 timing; in-repo caller first (T5-32) | 404 acceptable, no aliasing (FDS §21.4) | T5-22, T5-80 |
| `GET /rates/engine/quote` | exists (`:49`) | RE-POINT availability/blockReason to authority eligibility; pricing/hash retained | `use-crs-book.ts:87,129`, `front-office.api.ts:193` | after G1 | route stays; signal semantics change to codes | T5-14 (+see NB-4) |
| `GET /rates/engine/{availability,restrictions}` | exist | RETIRE | in-repo (scan) | after G1 | 404/strip | T5-23, T5-80 |
| `POST /rates/engine/{book,release}` | exist | RETIRE (book: after reads disposed; release: after writers gone) | `use-crs-book.ts` | strict order (P4) | revert-safe | T5-24, T5-25 |
| `POST /rates/engine/modify` | exists | legacy availability leg removed after gates | reservation modify | P4 (T5-50) | dual-write pin until then | T5-49, T5-50 |
| FO upgrade gate | `upgrade-room.handler.ts:69` legacy `isAvailable` **verified live**; `:85` assertion exists | authority eligibility consult; absent-row-true default deleted | front-office | after G1 | stricter gate, revert = legacy service retained | T5-13 |
| duplicate `GET /tax-rates` | declared twice: `availability-sales.controller.ts:131` **and** `banquet-refs.controller.ts:45` — **both verified live** | single owner after E-7 observation | — | E-7 first | no aliasing | T5-27, T5-90 |
| Error semantics | codes exist (`AVAILABILITY_ASSERTION_REJECTED` + `rejection.code`) | deterministic codes only; DTO validation before effect | FE + integrations | ungated | message-substring detection removed | T5-26, T5-34 |
| UNKNOWN/UNRESOLVED semantics | fail-closed live (verified) | pinned, never collapsed | all | ungated | — | T5-01/T5-03/T5-04 |
| Property isolation | authorize live | route wins over header | all | ungated | — | T5-19, T5-64 |

No API semantics require invention: current contract observed, target contract fixed by FDS §21/§10.2, order fixed by §24.2, retirement condition fixed by E-5 rule (timing only).

**Gate R7: PASS.**

## 16. Backend Consumer Readiness

| Consumer (Stage 1 §6 inventory) | Current authority | Target authority | Task | Dependency | Tests | Gate |
|---|---|---|---|---|---|---|
| Reservations write paths (create/status/cancel/no-show/reinstate/batch/waitlist) | assertion port (Phase 2) | unchanged — pinned | T5-12 | none | reservation integration specs | INV-P5-03/22/24 |
| Reservation modify leg | assertion **+** `crs.modifyReservation` (`repository:621/625` verified) | assertion only | T5-50 (+pin T5-49) | G1 + reads disposed + T5-53 | modify/extend integration + T5-81 | BLK-P5-03 |
| FO room upgrade | legacy `isAvailable` (`:69`) + assertion (`:85`) | authority eligibility + assertion | T5-13 | BLK-P5-01 | unit + scan | BLK-P5-01 |
| CRS quote signal | legacy `rate_restrictions` (`crs-engine.service.ts` `:149-163`) | authority eligibility codes | T5-14 | BLK-P5-01 | quote tests (pricing byte-identical) | BLK-P5-01 |
| CRS book path | raw `INSERT…'CONFIRMED'` | reservation create + assertion | T5-24 (+T5-31 FE) | P4 order | booking-through-create integration | BLK-P5-03 |
| CRS release / modify retire | legacy writers | retired / assertion-only | T5-25, T5-50 | writers gone | writer scan | §24.4 |
| GBA (pickup consult, cascade, aggregates) | own ledger + consult flag | unchanged (scope guard) | T5-15 | none | GBA battery (79 suites) | no behavior change |
| Allotments/blocks/wash | own ledger | unchanged (wash blocked) | T5-15, T5-54 docs | — | GBA battery | P-21 |
| Rates tab backend (`/rates/availability`) | unscoped legacy | retire/scoped | T5-22 | E-5 timing | route test | E-5 |
| Reporting/KPI | occupancy reporting + mislabelled figures | occupancy=declared reporting; availability-labelled→authority | T5-17 (+T5-35 FE) | G1 for authority values | payload tests | AC-42 |
| Channels outbound push | caller-supplied numbers | never claims authority | T5-16 | none | guard tests | AC-41 |
| Jobs/invalidation (`events.consumer`) | hotel-prefixed deletes | unchanged + verified | T5-67 | none | job tests | INV-P5-13 |
| Reconciliation readers / audit-log viewers / restriction editors | sanctioned reads | same, conformed routes | T5-18, T5-33 | T5-46/48 for editors | route contract tests | BLK-P5-02 for editing |
| Operational (logs, interval) | A3 handlers (raw) | authority write path (A3 = caller) | T5-45/46 | none to build; wired at T5-46 | handler tests + scan | DS-04 |
| Admin/mobile surfaces | none exist | none created | (M-7 recorded, T5-28 review) | — | — | scope fence |
| FO frontend quote caller (`front-office.api.ts:193`) | consumes `GET /rates/engine/quote` | route retained; signal re-pointed server-side | **implicit — see NB-4** | T5-14 | typecheck + T5-80 | non-blocking note |

No consumer remains accidentally dependent on a legacy authority after its assigned task; every MIX/LEG consumer from audit §6 has a task or an explicit "unchanged (scope guard)" verdict (GBA).

**Gate R6: PASS (with NB-4).**

## 17. Frontend Readiness (T5-29…T5-38)

| Task | Current consumer | Target API/authority | Query/state change | UNKNOWN/UNRESOLVED | Error/loading/empty | Legacy calc removal | Tests |
|---|---|---|---|---|---|---|---|
| T5-29 hook | none (E1: zero consumers) | `/availability/snapshot` via React Query | new hook, key includes propertyId/range | error codes exposed | error state mapped | n/a | hook unit |
| T5-30 page | `AvailabilityPage.tsx` (occupancy math `:851-876`, mislabel `:1006-1015`) | matrix projection + snapshot eligibility | data source swap | distinct UNKNOWN/BLOCKED render | label while OFF | **yes** — formulas deleted | component tests |
| T5-31 quick-book | `AvailableRatesMatrix.tsx:61-89` hardcoded `available:0` | projection cells + eligibility | cell mapping | UNKNOWN not bookable | blocked state shown | literal removed | mapping tests |
| T5-32 rates tab | `rates-inventory/page.tsx:143,155` | stop calling `/rates/availability` | call removed / figures from authority | n/a | n/a | n/a | T5-80 scan |
| T5-33 logs/interval | `reservation.api.ts:405/420/431` (**broken 404s verified**) | canonical routes + payload | route/payload conformance; edit affordances **disabled until T5-48** | n/a | no silent 404 (AC-45) | n/a | client tests |
| T5-34 errors | `use-crs-book.ts` + mutation hooks | `error.code` matching | substring detection removed | codes distinct | deterministic messages | n/a | per-code tests |
| T5-35 KPI | layout `:108-115`, KpiHeader, ChartsRow, reporting mock `78.4%` | authority-labelled / declared occupancy | payload sourcing | n/a | mock never as live truth | mislabelled formulas removed | payload tests |
| T5-36 GBA views | `GroupBookingsListView/DetailView`, `AllotmentDetailView` client math | server ledger fields | use server values | n/a | n/a | `quota-picked-released` etc. removed | component tests |
| T5-37 shadow stores | `reservationStore.roomTypeList`, `hotelStore ||200`, analytics fallback | authority values | fake defaults removed | n/a | no fake certainty | yes | static + tests |
| T5-38 tests | zero availability coverage today | new suites | — | covered | covered | covered | `apps/web && pnpm test` |

**ZERO UI/UX redesign confirmed:** plan §20 preamble + §36 non-goal 9 — only data sources, mandated surfacing states (label/UNKNOWN/BLOCKED), removed illegal content (client math, mock-as-live), disabled-not-404 affordances. No layout/copy/branding tasks. Frontend targets/behavior specified per task; no visual decision points.

**Frontend scope: DETERMINISTIC.**

## 18. Legacy Retirement Readiness

| Legacy source | Owner | Replacement | Dependency | Cutover/retirement condition | Verification | Rollback |
|---|---|---|---|---|---|---|
| A3 availability overlay (`:540`) | T5-40 | authority fact value | flag ON (T5-56) | ON window only | T5-20/T5-78 scans | guard removal is ON-branch only (OFF path untouched) |
| A3 raw restriction display reads (`:384-405`) | T5-39 | authority projection (T45/46 read side) | DS-04 read cutover | after T5-46 | scan (hotel-scoped today, verified) | revert to raw reads |
| `GET /rates/availability` | T5-22 | retired/stripped | E-5 timing; T5-32 first | in-repo caller gone | T5-80 | revert restores route |
| `/rates/engine/{availability,restrictions}` | T5-23 | retired | T5-14 + consumers switched | after G1 | T5-80 | revert |
| `/rates/engine/book` | T5-24 | reservation create + assertion | P4 order + E-5 | reads disposed | T5-80 + booking integration | revert (re-cutover forward) |
| `/rates/engine/release` | T5-25 | retired (writers gone) | T5-24 + T5-50 | writer set = ∅ | T5-81 writer scan | revert route only |
| Modify legacy leg (`:625`) | T5-50 | assertion port | G1 + DS-03 + checklist | BLK-P5-03 | dual-write pin flips; recon | **revert restores `:625`** (service retained) |
| FO `isAvailable` gate | T5-13 | authority eligibility | G1 | — | scan | revert (legacy retained) |
| Legacy `availability` table | P-16 | retained read-only | Phase 11 | never in Phase 5 | T5-41 (no new readers) | n/a (untouched) |
| Dead artifacts (`AvailabilitySales.tsx`, `operasales/**`, dead DI `reservation.repository.ts:15,89`) | T5-42 | replacements exist first | **G-12: hygiene last (P6)** | after all replacements | import-graph + suites | revert commit |
| Duplicate routes / doc drift | T5-43 | consolidation after consumers switched | G-12 | last | route scan | revert |

Legacy routes are **not** removed prematurely (retirement tasks all gated: E-5 / G1 / P4 / writer-set-empty); legacy writers **not** removed before canonical writes safe (T5-50 last, pin T5-49 active from start); retained read-only table behavior explicit (T5-41, P-16); hygiene tasks last (§8.3 rule 4).

**Legacy retirement: READY.**

## 19. Dual-Write / Cutover Readiness (DS-05)

| Element | Planned | Verified |
|---|---|---|
| Canonical writer | assertion port (`replaceReservationAssertion`, `:621-624` observed) | exists |
| Legacy writer | `crs.modifyReservation` leg (`:625` observed) | exists — dual state real |
| Dual-write requirement | **preserve** until gates (T5-49 pin from Stage 6 start; BR-5-026) | plan §18 |
| Read cutover | P1/P2 (T5-30/31/32, T5-14, T5-22/23) after G1 | §29 |
| Write cutover | T5-50 only after G1 + readers disposed + checklist | §29 P4 |
| Reconciliation | T5-57 read-only + drift shows modify-drift disappearing post-T5-50 | §21 |
| Mismatch handling | flagged deltas; drift monitored (F-04) | T5-57 |
| Flag behavior | `gba.a3.authoritative` ON only via T5-56 (4 conditions, evidence-recorded); never flips in code commits (G-5) | T5-55/56 |
| Retirement | T5-25 after writers gone | T5-81 |
| Rollback | revert `:625` (service retained to Phase 11); flag OFF; DI line revert; stub retained | §28 |
| Unsafe ordering possible? | **No** — §8.3 normative rules + T5-53 per-step checklist + T5-49 anti-cut pin | reviewed §25 |

**Gate R9: PASS.**

## 20. Feature Flag Readiness (DS-10)

| Aspect | Status |
|---|---|
| Single mechanism | env `FEATURE_*` via `ConfigService.getFeatureFlag` — **verified live** (`config.service.ts:63-65`: `FEATURE_${name}`, default `'false'`) |
| Flag inventory (7) | `gba.a3.authoritative`, `gba.consumers.cascade`, `gba.wash.schedulerEnabled`, `gba.reconciliation.enabled`, `gba.pickup.twoLayerConsult`, `gba.pickup.canonicalRead`, `gba.pickup.canonicalWrite` |
| Ownership | ops/env (operational); code reads only |
| Default state | OFF except documented deploy invariant (cascade ON at deploy — BR-5-038); T5-55 pins defaults |
| Enable sequence | a3.authoritative: 4-condition gate (T5-56); canonicalRead→soak→canonicalWrite untouched; wash blocked (Deviations A+B) |
| Disable behavior | flag OFF restores labeled legacy matrix view (decision-sanctioned rollback, FDS §24.7) |
| Dependency rules | §8.3 rule 5 (no flag-ON claims before T5-56 evidence) |
| Observability | flag state reported at every exit battery (`04` §34 item 9) |
| Removal condition | Phase 11/flags cleanup — not in Phase 5 (explicitly out) |
| Undocumented flags | none introduced (T5-84 scan); no platform-DB flags (T5-55) |
| **Flag values changed during Stage 5?** | **NO — zero flag changes this stage (verified: no config/source files modified)** |

Note (OBS-2): `compose.yaml` and `.env.example` currently contain **zero** `FEATURE_` lines (re-grepped this session) — confirming E3/audit finding and T5-54's documentation target exists as planned.

**Gate: DS-10 READY (flag governance non-blocking — BLK-P5-05).**

## 21. Logs / Audit / Reconciliation Readiness (DS-11)

| Element | Task | Verification planned | Present? |
|---|---|---|---|
| Canonical availability logs (GET+POST `/availability/logs`) | T5-33 (FE conformance) + T5-21 (DTO) | client route tests + DTO tests | yes (routes verified live) |
| Interval-update canonical payload | T5-33 | payload conformance test | yes |
| Audit events (append-only movements, identity/actor/hotel/date) | T5-58 | trigger rejection of balance UPDATE; insert-only; mutation→record | yes |
| Reconciliation read-only + flagged facts | T5-57 | zero-writes snapshot; authority wins downstream | yes |
| Mismatch detection | T5-57 | flagged deltas | yes |
| Legacy writer detection | T5-81/T5-41 scans | zero unauthorized writers post-gates | yes |
| Operational diagnostics | T5-18 | readers consume sanctioned outputs only | yes |
| Error classification | T5-26 | deterministic code set incl. `CONFLICT`/`OPERATION_IN_PROGRESS`, typed delete-reject | yes |
| Delete-reject-when-availability-state (BR-5-043) | T5-59 (+T5-60 never-releases pin) | rejection with **zero** row changes; no raw FK error; DB Restrict intent verified (`schema.prisma:17399/:17426`) | yes |
| Logs-never-authority guard | T5-61 | static scan | yes |

All required tests exist in the plan (AC-26/34/35/36 paths mapped in §27.1/§29 of plan).

**Logs/audit/reconciliation: READY.**

## 22. Test Readiness

| Level | Plan task | Scope | Critical coverage |
|---|---|---|---|
| 1. Unit | T5-69 | pins T5-01/03/08/55 + DTO + identity | AC-03…10, AC-12, INV-04…11 |
| 2. Database | T5-70 | 48 postgres suites w/ env; **0 env-skips** | AC-43, assertion/delete/scoping |
| 3. API contract | T5-71 | snapshot/matrix/recon/logs/interval/errors/isolation | AC-13…17, 26…29, 35 |
| 4. Web | T5-72 (+T5-38 new suites) | typecheck 0, lint 0, tests; baseline failure suite preserved as-is | AC-19/22/27 client |
| 5. GBA regression | T5-73 | 79-suite baseline; delta = investigation not redefinition | AC-11, GBA invariants |
| 6. Property isolation | T5-74 (WS-L suites grouped) | two-property fixtures across all paths | AC-13/14, INV-13/14 |
| 7. Concurrency/idempotency | T5-75 | replay/conflict, dual-write pin, assertion conflicts | AC-32/33 |
| 8. Static analysis | T5-77…86 (§23) | 10 scan families | prohibitions |

- Every AC and INV has a validation path (verified §9; e.g. AC-47←T5-11, AC-45←T5-33/48, AC-37←T5-55, AC-44←T5-62).
- Lightweight read-only verification only this stage: file counts re-verified (178/48/5 web test files) — **no suite executed, no test modified, no baseline "fixed."**
- Known baselines preserved: API 178/1470/1skip/6fail (2 suites/6 tests), web `reservation-actions.test.ts` 10 fails, sole skip `t45:293`.

**Gate R10: PASS.**

## 23. Static Analysis Readiness (T5-77…T5-86)

| Scan | Target | Method | Expected result | Failure condition | Completion gate |
|---|---|---|---|---|---|
| T5-77 legacy arithmetic | AvailabilityPage/KpiHeader/RoomGrid/GBA math/`hotelStore \|\| 200`, `rates-inventory.service.ts:34-40`, `inventory.domain-service.ts:96-117` | scripted rg/grep in tests | 0 hits reachable from new code | hit → fix or retirement note | every code exit |
| T5-78 duplicate authority | `hasZeroSell ? 0`, stray `sellableAvailable`, `available:` literals, new endpoints | scripted | 0 | hit → remove; registry test fails | every exit |
| T5-79 unscoped queries | availability/restriction SQL w/o `hotel_id` | scripted | 0 | hit → scope or revert | every exit |
| T5-80 legacy route usage (FE) | `/rates/availability`, retired engine routes, `/availability/matrix/logs`, `/activities/availability/*` | scripted | 0 after respective gates | hit → switch caller or delay retirement | after P1–P4 |
| T5-81 legacy writers | `$executeRawUnsafe` outside sanctioned service; engine/inventory writes post-gate; `INSERT INTO availability` | scripted | 0 post-gates (allow-list: `restriction-write.service.ts`) | hit → revert; **gate order violation = exit fails** | after T5-46/50/25 |
| T5-82 FE rule duplication | duplicate payload decls; client availability formulas | scripted | 0 | hit → adopt T5-02 types | every exit |
| T5-83 DB bypass / `body: any` | availability write routes; new raw SQL; raw FK errors | scripted | 0 | hit → DTO/service | every exit |
| T5-84 stale flags | platform-DB availability flags; undocumented `FEATURE_*`; code-level flag mutation | scripted | 0 | hit → remove/restore docs | every exit |
| T5-85 dead migrations | new files under `packages/db/migrations/**`; schema diff | scripted | 0 | **hit → hard stop** | every exit |
| T5-86 unsafe fallbacks | `hotelId \|\| 'default'`, `'default'` literals in availability paths, bare-id mutations, cross-hotel aggregates | scripted | 0 | hit → explicit rejection | every exit |

Implementation note in plan: scans run **inside `pnpm test`** (Phase 4 precedent T-31/T-39/T-59/T-65) with explicit allow-list files. All 10 families have target/method/failure/completion; "expected result" = clean scan (stated as scan-clean completion per row).

**Gate: static-analysis READY.**

## 24. Rollback Readiness

| Mechanism | Verified |
|---|---|
| Flag OFF | resting state = previous behavior (defaults OFF verified in config.service); T5-55 pins |
| DI revert | one binding line (`availability.module.ts:26` observed) + stub class retained (`unresolved-restriction.adapter.ts` exists; its spec `restriction.adapter.spec.ts` exists) |
| Stub retention | T5-09 keeps `UnresolvedRestrictionAdapter` present; T5-11 asserts existence pre-gate |
| Legacy retention | P-16: table + legacy services retained to Phase 11 ⇒ every code revert genuinely restores behavior (plan §28 global frame) |
| No data migration rollback | confirmed zero schema/data migration (§13) |
| Rollback triggers | per-switch table (plan §28, 14 switches) + flag-ON smoke failure ⇒ flag OFF |
| Post-rollback verification | §34 battery re-run; §28 note "any rollback executed is recorded in exit evidence" |
| Irreversible actions lacking safety boundary | **none found** — the only "one-way" element (T5-49 pin flip T5-50→single leg) is paired with git-revert + retained legacy service; T5-46 raw-SQL removal is behavior-preserving refactor with revert path; T5-42/43 hygiene are revert-commits gated last |

**Gate: rollback READY.**

## 25. Dependency / Execution Order Review

| Chain | Content | Audit result |
|---|---|---|
| CHAIN-1 | T5-05, T5-06 → T5-07 → T5-08 → T5-09 → T5-10 → (T5-56) | acyclic; evidence precedes build scope; binding swap requires all 4 records — **correct** |
| CHAIN-2 | T5-29 → T5-30 → T5-31 → T5-32; T5-40 rides T5-56 | after CHAIN-1 (§8.3 rule 1); T5-32 also depends on T5-22 disposition (P2 coordination, not circular: T5-22 has no dependency on T5-32) |
| CHAIN-3 | T5-13, T5-14 ∥ T5-22…25; T5-24 needs T5-14 + T5-31 | gated on G1 + E-5 timing — correct |
| CHAIN-4 | T5-49 → [G1 + CHAIN-3] → T5-50; T5-53 at every step | last, highest-risk flagged (B-4) — correct |
| CHAIN-5 | T5-45 → T5-46 → T5-48; T5-39 after 45/46; T5-47 independent | build ungated, wire gated — correct (BLK-P5-02) |
| CHAIN-6 | T5-87/88/89/90/76 evidence | independent, read-only — correct |
| CHAIN-7 | T5-42, T5-43 after replacements | G-12 last — correct |

Phases P0→P6 + G1 (plan §29): no task in P0 depends on a later phase; G1 chain strictly serial; P4 strictly serial; P6 strictly last. §30 parallelization: track file-sets disjoint (TR-1 evidence / TR-2 pins / TR-3 API / TR-4 restriction build / TR-5 FE ungated / TR-6 isolation+schema / TR-7 logs / TR-8 flags / TR-9 scans); shared files sequenced inside their phase (e.g., `availability-sales.controller.ts` touched by T5-21→T5-46→T5-39 landing order recorded in T5-53).

| Defect class checked | Result |
|---|---|
| Circular dependencies | **none found** (dependency edges all point to earlier/parallel ungated work) |
| Missing prerequisites | **none found** (spot-audited all gated tasks: each names its gate task(s)) |
| Unsafe parallelization | none (§30 conflict rule + disjoint file sets) |
| Task starting before evidence gate | none — evidence tasks first in P0; gated tasks name E-x |
| Task starting before blocker clearance | none — T5-07/09/13/14/24/30/31/35/40/50/56 all carry BLK-P5-01 or explicit evidence gates |
| Retirement before cutover | none — retirements ordered after reader migration (M-3/§24.2) |
| Cleanup before validation | none — T5-42/43 in P6, G-12 gated |
| Ordering inversion risk (DS-01→DS-03→DS-05) | protected by §8.3 + T5-49 pin + T5-53 checklist |

**Gate R11: PASS.**

## 26. Phase 4 Carry-Over Review

| Carry-over | Represented in `04` | Status preserved |
|---|---|---|
| **BLK-1 / Deviation A** (`GUARANTEED_BLOCK` wash exclusion; t45 spec skipped `:293`) | §32 row 1 + §6 baseline (sole skip) | open, Phase 4-owned; wash flag blocked; skip recorded not "fixed" |
| **BLK-2 / Deviation B** (durable wash/release/attrition store undecided) | §32 row 2 | open, Phase 4-owned |
| **Deviation C** (no schema changes) | §32 row 3 → reinforces G-1 → T5-62 | enforced |
| **Deviation D / NB-1…NB-6** (doc chores) | §32 row 4 → Stage 4/6 doc duties + T5-43 portion | carried as doc-only, never silently closed |
| **T-55** (Phase 4 test-maintenance) | §32 row 5 — Phase 4-owned, not re-planned | preserved |
| Cascade flag deploy configuration | §32 row 6 — documented (T5-54), operational | preserved |
| Canonical cutover leftovers | §32 row 7 — executed only via §29 under BLK-P5-03 | preserved |
| F-24 rule (Phase 5 may not assume Phase 4 done) | `04` §32 verification item + §34 exit checklist item 10 | enforced |
| DEF-3 pickup assertion gap (Phase 4) | T5-12/T5-15 document as inherited — explicitly **not** fixed | preserved, no invented fix |
| Phase 4 test baselines (178/1470/1skip/6fail; web 1-suite/10-test; GBA 79/766) | §6 + T5-76 reconciliation | never redefined |

No carry-over silently closed or reinterpreted.

**Gate: carry-overs PRESERVED.**

## 27. Readiness Matrix

| Area | Status | Evidence | Affected Tasks | Blocker | Required Action |
|---|---|---|---|---|---|
| Domain authority | READY | §8 (R1) | all | none | none |
| Stage 3 requirements | READY | §7 (116+6/116+6, 47/47, 34/34, 10/10, 51/51) | all | none | none |
| Business rules | READY | §8, §31.1 mapping | all | none | none |
| Restrictions (DS-01) | READY (gated) | §12 — fully specified, E-1/E-3 gates correct | T5-07…10, 13/14/40/56 | BLK-P5-01 (conditional) | execute T5-05/T5-06 first |
| Evidence gates | READY (open by design) | §11 | T5-05/06/76/87–90 | E-1/E-3 hard-open | record evidence; never assume |
| Database | READY (zero change) | §13 | T5-62/63 | none | run T5-62 at every exit |
| Property isolation | READY | §14 (authorize verified; unscoped query remediation named) | T5-64…68, T5-65/79/86 | none | execute scans |
| APIs | READY | §15 (routes/labels/404s verified live) | T5-19…28, 33 | E-5/E-7 timing only | — |
| Backend consumers | READY (NB-4) | §16 | T5-12…18 | BLK-P5-01 (13/14) | gated tasks wait |
| Frontend consumers | READY | §17 | T5-29…38 | BLK-P5-01 (30/31/35) | ungated half starts P0 |
| Legacy retirement | READY | §18 | T5-22…25, 39…44 | E-5 timing, G-12 | order per §29 |
| Dual-write/cutover | READY | §19 | T5-49…53 | BLK-P5-03 | checklist per step |
| Feature flags | READY | §20 (config mechanism verified; zero flags changed) | T5-54…56 | BLK-P5-05 (non-blocking) | document (T5-54) |
| Logs/audit | READY | §21 | T5-57…61, 21, 33 | none | — |
| Reconciliation | READY | §21 | T5-57 | none | — |
| Testing | READY | §22 | T5-69…76, 38 | BLK-P5-04 (evidence policy) | battery at exits |
| Static analysis | READY | §23 | T5-77…86 | none | scripted in tests |
| Rollback | READY | §24 | all modify tasks | none | — |
| Dependencies | READY | §25 (no cycles/inversions) | all | — | follow §29 |
| Phase 4 carry-overs | PRESERVED | §26 | none (non-actions) | BLK-1/2 Phase 4-owned | do not assume done |

## 28. Gate R1–R12 Results

| Gate | Result | Evidence |
|---|---|---|
| **R1 Domain Integrity** | **PASS** | §8 — no reopened decisions; semantics match source behavior (verified live) |
| **R2 Specification Integrity** | **PASS** | §7/§9 — 51/47/34/10/116+6 covered; 0 drops, 0 orphans |
| **R3 Evidence Integrity** | **PASS** | §11 — E-1…E-8 open/explicit; E-2 deferred-default preserved; nothing fabricated |
| **R4 Blocker Integrity** | **PASS** | §10 — 6 blockers classified per FDS; none closed/downgraded; isolation of gated work correct |
| **R5 Implementation Determinism** | **PASS (with notes)** | §5 of prompt question answered YES: no task requires an undocumented decision; format/citation notes NB-1…NB-3 are non-substantive |
| **R6 Consumer Completeness** | **PASS (with note)** | §16 — all audit §6 consumers tasked or explicitly scope-guarded; NB-4 (FO quote caller implicit) |
| **R7 API Readiness** | **PASS** | §15 — current/target/order/compat/tests fixed per endpoint; 404s and duplicates verified |
| **R8 Data / Isolation Safety** | **PASS** | §13/§14 — zero schema; existing-table reuse; unscoped query remediation named; authorize() verified |
| **R9 Cutover Safety** | **PASS** | §19 — pins, checklists, flag gates, rollback paths |
| **R10 Testability** | **PASS** | §22/§23 — every critical behavior has a validation path incl. scans |
| **R11 Dependency Safety** | **PASS** | §25 — no cycles/missing prerequisites/unsafe parallelization |
| **R12 Change Boundary** | **PASS** | §36 fence in plan; this stage changed only `05_READINESS_REVIEW.md`; Stage 6 scope explicitly fenced (no schema, no new deps, no redesign, no carry-over execution) |

**12/12 PASS.**

## 29. Findings Register

Classification: BLOCKER / CONDITIONAL BLOCKER / NON-BLOCKING NOTE / OBSERVATION.

| ID | Class | Finding | Impact | Handling |
|---|---|---|---|---|
| **BLK-P5-01** | CONDITIONAL BLOCKER | Evaluator + E-1 + E-3 open | blocks T5-07/09/10/13/14/24/30/31/35/40/50/56 only | Stage 6 may start with P0; closure = evidence (never assumption) |
| **BLK-P5-02** | CONDITIONAL BLOCKER | Restriction-edit UI wiring | blocks T5-48 + T5-33 affordance enable | interim disabled state defined (AC-45) |
| **BLK-P5-03** | CONDITIONAL BLOCKER | Cutover ordering | blocks T5-24/T5-50 | checklists + pin |
| **BLK-P5-04** | NON-BLOCKING NOTE | Executed-evidence policy | gates exits, not decisions | T5-76 |
| **BLK-P5-05** | NON-BLOCKING NOTE | Flag documentation/governance | docs task | T5-54/55 |
| **BLK-P5-06** | OBSERVATION (deferred by authority) | Out-of-scope items | no tasks (G-9) | §36 fence |
| **NB-1** | NON-BLOCKING NOTE | **Plan defect (exact text):** `04` §19 T5-40 Risk/Rollback contains an unresolved drafting artifact: *"revert restores overlay (flag OFF users unaffected either way — OFF path keeps label + legacy values until Phase 11? NO — matrix OFF path stays as interim view per P-17; overlay removal applies to the ON path only: implementation detail = guard removal inside ON branch)"*. The **settled meaning is determinate** (overlay removal = ON-branch only, consistent with P-17/§19 table), but the phrasing includes a rhetorical self-correction. | wording only; no decision needed (P-17 fixes the rule) | report; plan **not** edited in Stage 5; suggested clarity fix belongs to a Stage 6 doc-chore (T5-43 family) if at all |
| **NB-2** | NON-BLOCKING NOTE | **Path citation imprecision (exact locations):** (a) `04` T5-12 Scope says `apps/api/src/modules/reservations/repositories/reservation.repository.ts` — actual path is `apps/api/src/modules/reservations/infrastructure/repositories/reservation.repository.ts` (file+line refs `:621/:625/:889-924` all verified correct); (b) `04` T5-14 says `apps/api/src/modules/rates/crs-engine.service.ts` — actual is `apps/api/src/modules/rates-inventory/crs-engine.service.ts` (`:149-163` region verified in file). | locate-by-name works; zero decision impact | report; do not silently edit |
| **NB-3** | NON-BLOCKING NOTE | 26 of 90 tasks (T5-39…44, T5-69…76, T5-77…86 table form, T5-87…90) use compact/tabular field layout rather than the literal 9-field block; §11 of `04` claims all tasks carry the full set. Substance (authority cite, action, verification, failure/rollback, completion) is present; **no missing decision**. | format variance | agent executes from stated substance |
| **NB-4** | NON-BLOCKING NOTE | FO frontend quote consumer `front-office.api.ts:193` (+ doc-type `room-operation.dto.ts:28`) is inventoried in audit §10/finding F-07 but **not named in any T5 task's Scope/Files**; it consumes the retained (re-pointed) `GET /rates/engine/quote`, so server-side T5-14 + shared-type adoption (T5-02) + typecheck cover it, and T5-80 catches retired routes. | possible small extra file touch during T5-14/T5-34 | agent should treat quote-signal shape changes as covering all `engine/quote` callers found by scan/typecheck |
| **NB-5** | NON-BLOCKING NOTE | T5-35 offers two AC-42-compliant implementations for the `78.4%` mock ("mark as demo data" **or** "source it"); AC-42 constrains both (mock never presented as live truth). Minor presentation latitude, not a business decision. | cosmetic latitude | agent picks either; test asserts AC-42 either way |
| **OBS-1** | OBSERVATION | `04` §14 titled WS-D precedes §15 WS-C (section/workstream order mismatch) with an explicit numbering note in §14. | none | informational |
| **OBS-2** | OBSERVATION | `compose.yaml`/`.env.example` contain zero `FEATURE_` lines today (re-verified) — confirms audit E3 + T5-54 target; Stage 6 will add documentation (config-docs only, sanctioned by BR-5-036). | none | informational |
| **OBS-3** | OBSERVATION | A3 restriction raw reads at `availability-sales.controller.ts:384-405` verified **hotel-scoped**; the only unscoped availability computation found is `rates-inventory.service.ts:34-40` (named in T5-77/T5-22). | none | remediation confirmed present |
| **OBS-4** | OBSERVATION | `POST /availability/logs` uses `requireHotelId()` (header-derived) inside a `@PropertyScope(false)`-style controller — covered by T5-66 equivalence duty (route/header reconciliation). | none | per-plan handling |

**Totals: 0 BLOCKER · 3 CONDITIONAL BLOCKER (BLK-P5-01/02/03) · 7 NON-BLOCKING NOTE (incl. BLK-P5-04/05 + NB-1…5) · 5 OBSERVATION (BLK-P5-06 deferred + OBS-1…4).**

## 30. Required Preconditions

1. **E-1 (T5-05) and E-3 (T5-06) evidence recorded** before any BLK-P5-01-gated task starts — precondition for those tasks, **not** for Stage 6 start.
2. E-4/E-5/E-7 evidence tasks issued (T5-87/88/90); E-6 informative (T5-89); E-8 at first executed exit (T5-76).
3. Blockers BLK-P5-01/02/03 remain **open** until their stated conditions — a task existing does not clear a blocker.
4. Zero-DB assumption validated continuously by T5-62 (if a task is found to need schema work → stop + escalate, never silent migration).
5. Property isolation pattern (authorize() + route-wins + scans) applied to every touched path (T5-64…68).
6. Baselines frozen exactly as §6 of `04` (178/1470/1skip/6fail; web 1-suite/10-test; GBA 79/766/1skip/0fail; 51+2 migrations).
7. `AVAILABILITY_TEST_DATABASE_URL` available for DB suites (T5-54 documents; T5-70 executes).
8. This review (`05_READINESS_REVIEW.md`) present as Stage 6 handoff input.

## 31. Stage 6 Entry Conditions

Stage 6 **may begin** when:

- [x] Implementation Plan complete and structurally validated (§5)
- [x] Full traceability 51/47/34/10/116+6 (§7)
- [x] All blockers correctly classified and isolated — none bypassed (§10)
- [x] All evidence gates explicit, none fabricated; E-2 deferred-default preserved (§11)
- [x] DS-01 deterministic for its gated tasks (§12)
- [x] DB/schema assumptions validated as zero-change with gate task (§13)
- [x] Property isolation remediation identified incl. the known unscoped query (§14)
- [x] API contracts deterministic per endpoint (§15)
- [x] Consumer inventory complete (§16, NB-4 noted)
- [x] Frontend migration scope deterministic, zero redesign (§17)
- [x] Cutover sequence safe with pin + checklists (§19)
- [x] Rollback defined for every switch, no undocumented irreversible action (§24)
- [x] Tests defined for every critical behavior; scans defined (§22/§23)
- [x] Dependencies validated: no cycles, no inversions (§25)
- [x] Phase 4 carry-overs preserved, not assumed done (§26)
- [x] Change boundary intact: Stage 5 read-only (§32 report item 26)
- [ ] **First P0 action:** execute T5-05/T5-06 (evidence) — not a precondition for entry, but the critical-path first step

**Not required for entry:** E-1/E-3 completion (they gate their tasks, not the stage); GBA/wash decisions (Phase 4-owned); any schema work (prohibited).

## 32. Non-Blocking Notes

1. NB-1 — T5-40 rollback wording drafting artifact (settled meaning = ON-branch removal only).
2. NB-2 — two directory-level path citations imprecise (filenames/lines correct).
3. NB-3 — 26 tasks compact/tabular format vs §11's uniform-field claim.
4. NB-4 — FO quote caller not named in a task scope (covered by server re-point + typecheck + scans).
5. NB-5 — T5-35 two compliant mock-handling options (both satisfy AC-42).
6. BLK-P5-04/05 remain open governance items (by design; exit-evidence and flag-doc duties).

All notes must remain visible in Stage 6; none may silently alter scope. None requires rewriting the approved plan.

## 33. Final Readiness Verdict

**READY WITH NON-BLOCKING NOTES**

Rationale: the plan's 90 tasks are deterministic against Stage 2/3 authority and observed source behavior; every requirement is traced to a task and a verification; blockers and evidence gates are explicit, un-crossed, and correctly isolated so that Stage 6 can begin with P0 work without touching any gated path; zero-DB and isolation safety are verified with gate tasks; rollback paths exist for every switch. The five documentation-level notes (NB-1…NB-5) do not require any undocumented decision and do not change scope.

Not **READY** (bare) because: genuine open conditional blockers (BLK-P5-01/02/03) and open hard evidence (E-1/E-3) must stay visible, and five plan-documentation defects were found that a bare READY would obscure.
Not **NOT READY** because: no task requires an invented business/API/data/security decision, no orphaned requirement exists, and no unsafe ordering or irreversible action without safety boundary was found.

## 34. Stage 6 Handoff

**To Stage 6 (Implementation):**

1. Authoritative inputs: `01`, `02`, `03`, `04`, and this `05` (findings NB-1…NB-5 + OBS-1…4 visible).
2. Start at `04` §29 Phase **P0** (evidence tasks T5-05/06/87…90 first, then ungated pins/types/API/isolation/flags/logs/scans/tests). **Do not** start T5-07 before E-1; **do not** start T5-09 before E-1+E-3+T5-07+T5-08.
3. G1 serial gate closes BLK-P5-01 only with all five records; then P1…P6 per §29; P4 strictly serial; P6 strictly last.
4. Run the exit battery (`04` §34) at every code exit; first executed exit = T5-76 (E-8) with baselines explained, never redefined.
5. If any task reveals a need for a new business rule, schema change, new flag, or undefined API behavior → **STOP and escalate** (FDS §3.4 / §36). Never invent.
6. NB-1…NB-3 may be corrected only as explicit doc-chore work items if desired — never as silent edits.

**Stage 5 performed zero implementation:** no T5-task started; no source/schema/test/config/flag/database modified; git dirty count before = after = 776; only `docs/availability/phase-5/05_READINESS_REVIEW.md` created.

---

**PHASE 5 STAGE 5 — READINESS REVIEW COMPLETE**

**READY WITH NON-BLOCKING NOTES**




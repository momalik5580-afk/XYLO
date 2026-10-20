# PHASE 4 — IMPLEMENTATION PLAN EXECUTION CONTROL RECORD

**Phase 4 is a documentation-only planning phase. Phase 4 completion ≠ build authorization.**

Mandate: user-provided Phase 4 Implementation Plan Execution Mandate (conversation record, 2026-10-20). Mission: convert the locked Phase 3 Final Domain Specification (FDS) into sequenced, owned, gate-governed build work. Documentation only; zero implementation; no build activity authorized.

---

## 1. Mission and boundary

- Convert the Phase 3 FDS into workstream-plan documents for future build phases.
- **Documentation only.** No production code, test code, schemas, migrations, APIs, UI, configuration, dependencies, or operational artifacts were created or modified.
- No implementation tests executed. No commits, pushes, publishes, stashes, resets, checkouts, merges, rebases, remote writes, or state-changing Git commands. No fetch this session.
- Read-only source inspection was used to map workstreams to real implementation touchpoints.
- **Phase 4 completion ≠ build authorization.** This distinction is stated per-deliverable and per-workstream (01, 02, 03).

## 2. Repository facts (verified at session start, before any write)

| Item | Verified value |
|---|---|
| Repository root | `C:/Users/Pro/Desktop/XYLO` |
| Remote origin | `https://github.com/momalik5580-afk/XYLO.git` |
| Branch | `main` |
| HEAD | `ed278bc3a6817351c5b358292109828c412f365c` |
| HEAD commit | 2026-10-09 00:19:52 +0300, author `msmfx` — "fix(db): C-03 - recover reproducible migration baseline (GAP-06/ARCH-010/044/045)" |
| `git status --porcelain` (default mode) | **891 lines** (complete output captured read-only in session log; category counts below) |
| Tracked modified/deleted/untracked mix | **661 tracked M/D lines + 230 collapsed untracked lines = 891** |
| `git status --porcelain -uall` | **2,526 lines** |
| `origin/main` remote-tracking ref (local view) | `70f0616f6c1506b9790d967423e3cf36c3eef341` — pre-existing state from the fetch disclosed in the Phase 3 correction report; **no remote commands were run this session**; remote `main` is the owner's separate documents-mirror history (no common history with this repo) |

Relevant pre-existing working-tree facts for the Housekeeping module (not agent changes):

- `M apps/api/src/modules/housekeeping/housekeeping.controller.ts`
- `M apps/api/src/modules/housekeeping/housekeeping.module.ts`
- `D apps/api/src/modules/housekeeping/dto.ts`, `D apps/api/src/modules/housekeeping/roles.guard.ts` (tracked-file deletions, pre-existing user restructure)
- Untracked restructure layers: `apps/api/src/modules/housekeeping/application/`, `domain/`, `events/`, `infrastructure/` (pre-existing user work)
- Untracked docs of the housekeeping program: `docs/audit/Housekeeping/`, `docs/housekeeping/phase-2/`, `docs/housekeeping/phase-3/`

## 3. Baseline comparison and reconciliation

- **Recorded baseline HEAD `ed278bc3…` == actual HEAD: VERIFIED MATCH.** No branch movement.
- Working tree at session start: default-mode total **891** (661 tracked M/D + 230 collapsed untracked), `-uall` **2,526** — identical to the last recorded measurement (2026-10-20).
- Historical claim preserved per mandate §2.1: what the "pre-existing 891" was **before** the Phase 3 session began cannot be independently re-proven (1-line reconciliation gap documented in the Phase 3 correction report — the phase-3 write-set itself adds one collapsed untracked line). Status: **UNRECONCILED historical measurement note; carried as-is; no repository state was manipulated to force reconciliation.**
- The full default-mode status output was captured read-only this session (complete captured status per mandate §2.1; category counts verified; raw capture retained in session log).

## 4. Input-set verification (§2.3)

| Set | Expected | Verified | Method |
|---|---|---|---|
| Phase 3 — `docs/housekeeping/phase-3/` | 6 files | **6 files present** (00 control 9,811 B; 01 FDS 31,448 B; 02 traceability 8,455 B; 03 ODIs 6,563 B; 04 gates 6,192 B; 05 exit assessment 6,367 B) | Directory listing + `git hash-object`: all six blob SHAs **match the recorded final values exactly** (00 `0f11f2fc…`, 01 `779c4662…`, 02 `b959fdc2…`, 03 `4c50e605…`, 04 `cba02310…`, 05 `ded180d1…`) → **integrity VERIFIED, no drift since Phase 3 closure** |
| Phase 2 — `docs/housekeeping/phase-2/` | 7 files | **7 files present** (01 11,736 B; 02 register 15,434 B; 03 analysis 49,186 B; 04 consistency 13,020 B; 05 queue 21,246 B; 06 exit 8,841 B; 07 closure log 35,359 B) | Directory listing; content reads this session: `04 §7` (entry conditions, lines 127–131), `07 §9` (OQ-12 matrix, lines 170–216), `07 §10` (HKD-32 runtime record) |
| Phase 1 — `docs/audit/Housekeeping/` | 7 files | **7 files present**: `00_AUDIT_CONTROL.md`, `01_EXECUTIVE_AUDIT_REPORT.md`, `02_FINDINGS_REGISTER.md` (36,962 B — HK-001…HK-037), `03_WORKFLOW_MAPS.md`, `04_DOMAIN_AND_DATA_MAP.md`, `05_VERIFICATION_AND_COVERAGE.md`, `06_PHASE1_EXIT_ASSESSMENT.md` | Directory listing; findings cross-walk carried via `phase-2 07 §7` |

Remote mirror context (§2.4): recorded as history only (`origin/main` mirror at `70f0616f…`; stale published copies of phase-3 02/04/05 already reported to owner in the correction report). **The remote mirror is not an authoritative input for Phase 4 and was not touched.**

## 5. Phase 2 §7 entry-condition assessment (verified against `04_CROSS_DECISION_CONSISTENCY_AND_ORDER.md` lines 127–131)

| # | Recorded condition (quote/paraphrase) | Status | Phase 4 effect |
|---|---|---|---|
| EC-1 | "Ratification bundle (blocks implementation authority, not spec drafting): owner ratify/veto the 6 USER_DELEGATED resolutions (HKD-04, 06, 20, 21, 23, 30) and the 2 provisional defaults (HKD-10, 22); security-owner + owner explicit sign-off on the completed role→action matrix (OQ-12, 07 §9)." | **PENDING — acknowledged.** Not blocking Phase 4 *planning* by its own recorded scope; blocks future *implementation authority* | All delegated/provisional-anchored work is planned with `PENDING RATIFICATION` labels (§4.6); no label upgraded |
| EC-2 | "Mirror family (HKD-32): census complete — static + runtime (07 §3, §10); disposition = RETAIN as compatibility mirror; removal proposal staged with prerequisites for the later DB-phase gate; no Phase 2 removal authorized." | **SATISFIED as carried state** — disposition RETAIN unchanged | Phase 4 preserves: no new mirror consumers (F7.3); fail-visible approval leg precedes any mirror-leg consideration (F5.3); HKD-32 removal outside Phase 4 (F7.4) |
| EC-3 | "Conditional reopen guards (none spec-blocking): HKD-10 (shift evidence), HKD-22 (laundry model), HKD-12 (workload formula), OQ-09 (headcount evidence), OQ-14 (supplies-view scheduling)." | **SATISFIED (no trigger)** — none fired | Carried in gate map (Gate 4); no Phase 4 workstream depends on reopening them |

No entry condition was represented as satisfied beyond what its own record states.

## 6. Errata and discrepancy register

*Errors in Phase 1–3 records are NOT corrected in their files; they are documented here only (mandate §3.1).*

| ID | Finding | Evidence | Effect (workstream → consequence) |
|---|---|---|---|
| ERR-A | Phase 2 `07 §9.1` cites `roles.guard.ts:14` without a full path; ambiguity between the module-local guard and the common guard was resolved this session: the citation target is `apps/api/src/common/authorization/roles.guard.ts` (**verified present**); the module-local `apps/api/src/modules/housekeeping/roles.guard.ts` is a pre-existing tracked-file deletion in the working tree (user restructure) | `glob apps/api/src/**/roles.guard.ts` → 1 hit (common); `git status` shows module-local `D` | Informational; citation target clarified here only. Affects WS9 traceability only. |
| ERR-B | The engineering-maintenance module (owner of the OOO/OOS lifecycle per HKD-04/F9.1) contains **3 files only**, and a bounded grep for `OUT_OF_ORDER|out_of_order|OOO` found **zero matches**: the in-code OOO/OOS lifecycle owner capability is not located in that module today | `Get-ChildItem apps/api/src/modules/engineering-maintenance` → 3 files; grep → 0 matches | Not a contradiction of Phase 3 (the FDS records ownership as a domain rule, not an in-code fact). **WS7 is therefore planned against an owner capability whose in-code realization is UNVERIFIED** — blocked-on input added; no maintenance contract invented (mandate WS7). |
| ERR-C | Historical "891 pre-existing paths" measurement: as-observed value today equals 891 including the phase-3 directory as one collapsed untracked line; the pre-Phase-3-session count cannot be independently re-proven (1-line gap) | §3 above | Limitation disclosure only; no state manipulation. |
| ERR-D | `moveRoom` (F4.8 "service method exists unrouted") — absence of a move route re-verified this session against the full 62-route capture: **no move route exists** in `housekeeping.controller.ts` (routes verified 31–409) | Route capture this session | Supports WS4 scope as specified (integration wiring is future work); no Phase 1–3 correction needed. |
| ERR-E | The recorded `linen_inventory` resource label (HKD-21/F11.1) was verified at behavior level this session (stock reads `quantity_on_hand`/`unit_cost` in `housekeeping-inventory.service.ts:72–80,128,235–236`); **exact linen table-name mapping in current schema was not re-established this session** | Grep verified read paths, not table names | WS6 design must re-verify the table-name mapping against the Prisma schema before implementation; marked as a WS6 design-verification item (UNVERIFIED detail inside a verified capability). |

No other discrepancies found. Anchor integrity of Phase 1–3 citations checked in current code: `housekeeping.service.ts:59–60` (HK-005 ids), `:162/165–174` (HK-004), `:560/577/582/584/590/593` (HK-006/-008/HKD-27), `:614–626` (HK-030), `:806/816` (HK-011), `:288–310` (HK-012), `:920–953` (HK-018), `:903–917` (write tolerance), `:353–355` (HK-034), `:83/152–159` (HK-026 matrix mapped), `check-out.handler.ts:471` (HKD-14), `room-readiness.spec.ts` (HKD-05 anchor), `get-room-detail.query.handler.ts` (HKD-32 prerequisite 2), `GuestRequestsSection.tsx:155/387` (HK-030), `LinenSection.tsx:20/46/57–58` (HK-002/HK-035 synthesized vocab), `DashboardSection.tsx:336` (HK-022 banner) — **all resolve; no stale anchors detected**.

## 7. Gates 1–6 carry-forward (verified against `phase-3 04_EXTERNAL_GATES_AND_CONDITIONS.md`, headings lines 10–42; rule anchors per `01_FDS_SYNTHESIS.md`)

| Gate | Identity & requirement (as recorded) | Source | Current status | Evidence | Effect on Phase 4 planning | Effect on future build | Owner (recorded) | Release evidence | Blocks… |
|---|---|---|---|---|---|---|---|---|---|
| G1 | Phase 3 entry **ratification bundle** — owner ratify/veto HKD-04, 06, 20, 21, 23, 30 (USER_DELEGATED) + HKD-10, 22 (PROVISIONAL) | `04` Gate 1; `02 §7` EC-1 | **PENDING (not executed)** | `04 §7` line 129; `07 §9` OQ-12 registry | Planning proceeds **around** delegated/provisional rules; affected work is labeled `PENDING RATIFICATION` (§4.6) | Bundle must be executed before those rules' implementation authority exists | Owner (product) | Owner's recorded ratification/veto acts | WS4 (office/collaboration parts F6.4–F6.6), WS5, WS6 capability scope analysis, WS7 — see bundle matrix in `02` |
| G2 | **OQ-12 explicit sign-off** — security owner + owner on the role→action matrix (07 §9) | `04` Gate 2; `07 §9.4` | **OPEN** | `07 §9.4`; F14.4/F14.5 | WS9 is planned but stays `BLOCKED`; no propagation to other workstreams (§4.2) | Hard-gates WS9 only (registration/decorator plan/role grants/inspector role) | Security owner + product owner (**non-delegable**) | Written sign-off evidence on 07 §9 matrix + the four F14.5 determinations | WS9 only |
| G3 | **HKD-32 mirror-family removal execution** — later DB-phase gate; four prerequisites (F7.4): approve mirror write retired/re-terminated; FO `room_status_history` read re-scoped; RLS-with-zero-policies resolved; divergence re-check | `04` Gate 3; `07 §10.4` | **RETAIN — removal deferred** (scheduled, outside Phase 4) | `07 §3/§10`; F7.1–F7.5 | No new mirror consumers; fail-visible (F5.3) before any mirror-leg consideration; HKD-32 removal remains outside Phase 4 | Removal executes only at its own later DB-phase gate | Owner with later DB-phase gate per staged prerequisites | Phase-gate evidence per F7.4 items | HKD-32 removal path only; sub-scope gates on WS3's mirror leg |
| G4 | **Conditional reopen guards** — HKD-10 (shift evidence), HKD-22 (laundry model), HKD-12/OQ-07 (workload formula), OQ-09 (headcount), OQ-14 (supplies-view) | `04` Gate 4 (F1.1–F1.6, F11.3, F1.5) | **No trigger on current evidence** (none spec-blocking) | `02 §7` EC-3; `07 §7` cross-walk | None; these reopeners are external-facts guards, not Phase 4 inputs | If evidence arrives, the named decision is reopened in the funnel (HKD-19 family) — separately gated | General Manager / hotel-ops channel as recorded | New operational facts (only if they occur) | Nothing currently |
| G5 | **Deferred-to-implementation register** — contract-level rules whose realization detail is owned later; Phase 4 consumed the later-phase-owner column of `phase-3 02 Part B` for touchpoint/owner mapping | `04` Gate 5 | **Active register** (consumed in planning) | `phase-3 02` Part B; all `Fdescriptions` marked "later phases/phases' own gates" | Phase 4 mapped every such item to its workstream under its recorded owner | Realization must follow the recorded rule text exactly | Per-row owners (recorded in `phase-3 02` Part B) | Realization with the recorded acceptance test set | Per-item |
| G6 | **FO co-sign constraint** — comparator re-anchored vacant-state set (ODI-04/HKD-17/18) requires FO co-sign **before that comparator wave's coding**; applies to same-wave rule (F7.2) | `04` Gate 6; `02 §7` (HKD-06 line), `07 §4` | **PENDING (co-sign not yet given)** | ODI-04 row (`phase-3 03` line 15) | Carried as a cross-module dependency; Phase 4 does not decide the set | **Blocks** the FO comparator re-scope *coding* only (not other waves); HK planning side carries the boundary note only | FO domain owner | Recorded FO co-sign on the re-scoped vacant-state set | Sub-scope of the FO comparator re-scope work; **no general workstream gate** |

Gates 4 and 5 were fields-verified directly against the `phase-3 04` register rows (mandate §6 requirement); no invented meanings.

## 8. Carried governance states (unchanged)

- `OQ-12 = OPEN` — security owner + product owner sign-off pending (`07 §9.4`; F14.5's four determinations: (a) `housekeeping:*` registry defs/grants, (b) route decorator plan for the 62 routes, (c) guard-system choice `@Permissions` vs `@Permission`, (d) inspector-role determination). **Blocks WS9 only.**
- Ratification bundle = **PENDING**: HKD-04, HKD-06, HKD-20, HKD-21, HKD-23, HKD-30 (USER_DELEGATED) + HKD-10, HKD-22 (PROVISIONAL). No label upgraded anywhere in Phase 4 deliverables.
- `HKD-32 = RETAIN`. No removal-safety claim is made anywhere in Phase 4 deliverables. Removal = later DB-phase gate with the four prerequisites of F7.4.

## 9. Evidence and source-authority rules applied (§4.6–4.8)

- Every HK finding (HK-001…HK-037), HKD record, ODI (ODI-01…ODI-12), Gate (1–6), and code touchpoint cited in Phase 4 is verified: identifiers against `phase-1 02_FINDINGS_REGISTER.md` / `phase-2 02` / `phase-3 03` / `phase-3 04` (existence cross-verified in `phase-2 07 §7` and `phase-3 02`); titles/labels from the authoritative texts; live code touchpoints by read-only inspection this session with `path:line` anchors.
- Workstreams that include a PROV/DELEGATED rule in scope are planned with `PENDING RATIFICATION` applied **only** to that rule's sub-scope; APPROVED/EB anchors proceed on their own authority.
- Verification integrity vocabulary is used consistently: `VERIFIED (this session, read-only)` / `UNVERIFIED (register-carried; inspections pending)` / `PENDING DECISION` / `BLOCKED` / `PROPOSED`. Nothing was executed: no implementation tests, migrations, schema ops, or runtime checks were run in Phase 4.
- Register-carried evidence (e.g., HK-022 full vulnerability enumerations, HK-001 annex tables, supply-transition dead-end HK-037) is cited to its record with the label `register-carried`; where Phase 4 also verified the cited anchor live this session, both are noted.

## 10. Preservation evidence and exact Phase 4 write-set

*Before-state verified this session (§2). After-state is stated in the Phase 4 execution report and re-checkable by diff; no pre-existing work was cleaned, reset, stashed, staged, or altered.*

**Authorized write-set (exclusively):** `docs/housekeeping/phase-4/` — exactly four files:

1. `00_PHASE4_CONTROL.md` (this file)
2. `01_PHASE4_IMPLEMENTATION_PLAN.md`
3. `02_DEPENDENCY_AND_GATE_MAP.md`
4. `03_PHASE4_EXIT_CRITERIA_PROPOSAL.md`

**Verified after-state (deltas as measured):** default-mode total remains **891** — no default-mode line was added, because `docs/housekeeping/` contains no tracked files, so Git collapses the whole parent directory into the single pre-existing line `?? docs/housekeeping/` (its scope expanded from 13 files — 7 phase-2 + 6 phase-3 — to 17 files by the 4 Phase 4 deliverables; `-uall` count grew exactly +4: 2,526 → **2,530**). HEAD unchanged (`ed278bc3…` re-verified after writing). Tracked M/D count unchanged (661). All six Phase 3 blob SHAs re-verified identical post-write; Phase 2 register/closure-log/consistency files re-hashed too. No Phase 1/2/3 file modified.

If an unexpected out-of-scope change is ever detected, mandate §3.3 procedure (stop, report, no self-repair) governs.

## 11. Mandate-directives compliance

| Directive | Compliance | Evidence |
|---|---|---|
| §3.1 write-set | SATISFIED | §10; four files only |
| §3.2 zero implementation | SATISFIED | No code/test/schema/config artifacts touched; §7.2 of mandate honored |
| §3.3 repo & remote preservation | SATISFIED | HEAD unchanged; no commits/push/publish/remote ops; before/after capture |
| §3.4 carried states | SATISFIED | §8 verbatim labels preserved |
| §3.5 three-way semantics | SATISFIED | §13; per-deliverable statements |
| §4.1 exit criteria = proposal | SATISFIED | `03` banner + verdict rules |
| §4.2 OQ-12 limited block | SATISFIED | WS9 `BLOCKED`; no propagation (01 §10; 02 graph) |
| §4.3 retirement own gates | SATISFIED | WS10; F1.10/F11.4 rule text preserved as recorded |
| §4.4 mirror restrictions | SATISFIED | WS3 sequencing constraint; F7.3 rule text; no new consumers planned |
| §4.5 blocked-on mandatory | SATISFIED | Every WS carries a blocked-on table (01) |
| §4.6 labels carry forward | SATISFIED | §9; registry in 02 §4 |
| §4.7 traceability | SATISFIED | §9; per-WS anchor tables; ERR-A–E |
| §4.8 verification integrity | SATISFIED | §9; per-WS evidence vocabulary |
| §4.9 out-of-scope absolutes | SATISFIED | WS plan exclusions; F1.1–F1.8, F2.4, F15.1, F-25-family rules preserved as recorded |

## 12. Deliverable index

| File | Purpose |
|---|---|
| `00_PHASE4_CONTROL.md` | This control record (mission, repo facts, inputs, entry conditions, errata, gates, carried states, compliance) |
| `01_PHASE4_IMPLEMENTATION_PLAN.md` | The ten fixed workstreams WS1–WS10 with verified anchors, touchpoints, future acceptance tests, entry/exit, blocked-on tables; wave sequencing |
| `02_DEPENDENCY_AND_GATE_MAP.md` | Blocking graph; gate/ODI/bundle/ratification matrices; ODI-01…ODI-12 decision matrix (exact recorded questions/owners/defaults/status); cross-module dependencies |
| `03_PHASE4_EXIT_CRITERIA_PROPOSAL.md` | **PROPOSED** Phase 4 exit criteria — owner ratification required; verdict rules; sign-off block |

## 13. Three-way status statement

1. **Phase 3 specification authoring:** `COMPLETE` — according to the existing Phase 3 record (`05_PHASE3_EXIT_ASSESSMENT.md`). Phase 4 does not reopen or alter it (blob integrity re-proven §4).
2. **Phase 4 plan completion:** assessed **only** against the criteria proposed in `03` — **owner ratification pending**; the criteria are mandate-provided proposals, not pre-approved.
3. **Build authorization:** `NOT GRANTED` by Phase 4 completion. Any future implementation requires a separate owner act plus satisfaction of the applicable workstream gates carried forward in `02`.

## 14. Statement

**Phase 4 completion ≠ build authorization. Documentation only. Nothing in the Phase 4 deliverables starts a build, authorizes a test run, or modifies the repository state.**

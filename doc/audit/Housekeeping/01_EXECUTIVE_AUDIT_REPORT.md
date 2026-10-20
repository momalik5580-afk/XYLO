# XYLO HOUSEKEEPING — EXECUTIVE AUDIT REPORT (DELIVERABLE 1)

Document: 01_EXECUTIVE_AUDIT_REPORT.md
Lifecycle step: Phase 1 — Forensic Audit
Status: COMPLETE
Date: 2026-10-20
Evidence basis: on-disk working tree at commit `ed278bc3` (`main`), repo substantially uncommitted — see 00_AUDIT_CONTROL.md §1. All claims are source-inspection evidence; runtime status is tracked in 05.

---

## 1. Executive summary

The Housekeeping module is a **read-heavy, write-fragile orphan**: most of its data surface works by scrolling pre-existing reference data, while nearly everything that *changes state* is either unwired, unguarded, or quietly self-degrading. Four core services, an uninstrumented CQRS command stack and a 62-route controller stand behind a frontend that cannot reach ~40 of its declared endpoints; the only feature built on the modern CQRS stack is broken end-to-end and has a ratified "never-created table" defect behind it. Rooms re-enter service through exactly one guarded-life path (inspection approval), and none of the module's scores of state mutations are tested anywhere in the repository.

The three statements below, with their register IDs, are the core of this audit:

1. **The frontend|backend contract is broken at scale, and silently.** ~40+ frontend calls (all linen writes, all laundry, supplies, teams, guest-request persistence, move-room, lost-found claim, discrepancy reads) target endpoints that do not exist server-side; store errors are not user-rendered, so the workflows "complete" and then silently lose everything on refresh ([HK-001]). One prominent flow is worse than a silent no-op: daily linen refill is *fabricated client-side* and never persisted ([HK-002]).
2. **The state machine that must drive a hotel PMS has no enforced spine.** The module owns housekeeping task status, room condition, inspection, and supply request state, yet only one of those (supply requests) has an enforced transition matrix — and its approval chain provably dead-ends ([HK-026], [HK-037]). Room condition is a split-brain across two column-sets with exactly two production writers beyond reservations' room-move, and no writer for CLEAN / OOO / OOS / front-office status at all ([HK-006], [HK-013]–[HK-016], [HK-019], [HK-020]).
3. **Authorization and audit-identity are present in the data model, absent at the gates.** The 62-route controller carries zero role/permission gates ([HK-007]) and persists client-supplied inspector identity — which, in the live UI, is an empty string ([HK-008]).

Everything else in the register (37 findings: 13 confirmed defects, 10 design-architecture risks, 3 UX gaps, 6 inconsistencies, 4 enhancement candidates, 1 open question cluster) is a consequence of, or contributor to, those three skeletons.

## 2. Audit scope and basis (short form)

| Item | Value |
|---|---|
| In scope | `apps/api/src/modules/housekeeping/**` (all); `apps/web/app/(dashboard)/housekeeping/**`; `apps/web/store/housekeepingStore.ts`; `apps/web/features/housekeeping/**`; HK persistence (schema + migrations + baseline declarations); HK-facing cross-module seams (front-office, room-service, lost-found, employees, command-center) |
| Out of scope | FO/room-service/lost-found functional audits (only at HK seam level); UI redesign; business-rule decisions; target-domain spec |
| Method | 6 parallel evidence tracks (frontend, API contract, room-status semantics, checkout integration, tests, permissions/tenancy), reconciled + lead-verified; no runtime execution (see 05) |
| Prior lineage | `docs/reservations/phase-3/01_R8_CLASSIFICATION_LEDGER.md`; `packages/db/migration-baseline.declarations.json:89–99`; phase-1 closure; OPERA gap analysis (documents only) |

Full scope table: 00 §2. Deliverable index: 00 §5.

## 3. What the system actually is (verified shape)

| Metric (VERIFIED FACT) | Value | Registered consequence |
|---|---|---|
| Backend routes | 62 (`housekeeping.controller.ts`, no class method count dispute) | 62 × 0 role gates → HK-007 |
| Backend services | 4 core (`service`, `inventory`, `stock`, `supply-request`) + a CQRS command stack for one feature | CQRS misapplication → HK-003 |
| Legacy frontend store | 1,523 lines, one Zustand store, all call sites | Store sprawl → HK-001 |
| Web components | 12 `_components`; 6 live, 6 dead (one crash-if-rendered) | HK-023 |
| React Query features | 1 (`features/housekeeping` person-discrepancy) — unmounted, mismatched | HK-003 |
| Web API clients | 2 different base URLs (`/api` legacy, `/api/v1` features) | HK-001 cross-client skew |
| Dead frontend endpoints | ~40+ | HK-001 |
| Room-condition writers in whole API | 3 (checkout DIRTY; HK approve INSPECTED + mirror, both failure-swallowed; reservations room-move) | HK-006/HK-013–016 |
| Writers for CLEAN/OOO/OOS/front_office_status | **0** | HK-014/HK-015/HK-019/HK-020 |
| Scheduler for "daily" tasks | none (manual endpoint only) | HK-004 |
| Module tests | **0** | 05 |
| Peripheral tests | 16 (12 API + 4 web), none pinning HK writes | 05 §2.2/§3 |

## 4. The five load-bearing facts (each detailed in the register)

### 4.1 Workflows that "succeed" without persistence (HK-001/002/003/022/023/026/030/037)
Detail: 02 §A rows 1–3, §C rows 22–25; 03 §2/§3/§6/§7/§8/§9. Reader-shaped data (staff, rooms, sections, outlets, inspections, linen/laundry reads, checks) mostly works; writes beyond the inspection and task-juggling flows mostly do not, and the store's error path ensures the operator cannot tell the difference.

### 4.2 One guarded gate keeps rooms entering service; the world after the gate is unowned (HK-006/013–016/019/020)
Rooms are re-entered only via inspection approval, which dual-writes `rooms.housekeeping_status` and a legacy `room_status` mirror — each write individually failure-swallowed (`.catch(() => {})`). No production path sets CLEAN, OOO, OOS, or `front_office_status` anywhere in the API (`housekeeping.service.ts:560–680`, `check-out.handler.ts:470–481`, FO gate `room.aggregate.ts:30` matching `CLEAN|INSPECTED|READY`). The `room_status` mirror is a fourth column-set the model carries through a disagreement about who owns the data (HK-013).

### 4.3 Daily generation is manual, destructive, and one-shot per room (HK-004/009/031/036)
`generateDailyTasks` (no scheduler is wired to it) resets **all** non-guest tasks to pending — *including already inspected/completed ones*, collapsing every operator's prior day of assignment progress — records **no audit entry** for that reset (the rest of the service does), skips any room that already has any task (no `business_date` column exists to make "daily" meaningful), and aborts the whole run on a single unique-key collision.

### 4.4 The CQRS person-discrepancy feature is broken end-to-end and sits behind a ratified schema defect (HK-003)
The only module feature built with CQRS: `GET /housekeeping/discrepancies` 404s (route absent), the React Query payload keys mismatch (`personCount`/`hkPersonCount` etc.), the backend `person_discrepancies` table was *never created* — ratified as `acceptedKnownDefect` in `packages/db/migration-baseline.declarations.json:89–99` — and the feature is mounted on no page anywhere in the app tree. The person-count workflow that still runs end-to-end does so only in the legacy string-protocol path between two UIs (03 §7). Detail: 02 §A row 3; 04.

### 4.5 Authorization/audit-attribution gaps concentrated at the write-path perimeter (HK-007/008/010/011/012)
Zero role/permission gates on all 62 routes (any authenticated tenant user can mutate all HK state); inspector identity is client-supplied — the live UI literally posts an empty string; and in the reassignment flow a supervisor id is written over an attendant assignment while the assignment record itself overloads a JSON column (HK-011).

## 5. Findings summary (counts + register pointer)

| Category | Count | Register pointer |
|---|---|---|
| [D] Confirmed defects | 13 (HK-001..HK-012, HK-037) | 02 §A + §F |
| [R] Design/architecture risks | 10 (HK-013..HK-021, HK-024) | 02 §B + §C row 24 |
| [G] UX/functional gaps | 3 (HK-022, HK-023, HK-025) | 02 §C |
| [I] Inconsistencies | 6 (HK-026..HK-031) | 02 §D |
| [E] Enhancement candidates | 4 (HK-032..HK-035) | 02 §E |
| [Q] Unverified/open | 1 (HK-036 cluster) | 02 §E/§4 below |
| **Total** | **37** | Annex B cross-walk |

Severity concentration: the highest-consequence [D] findings are the three skeletons of §1; the remainder concentrate on identity-collision exposure (HK-005, HK-009) and broken joins/attribution (HK-012). Of the [R] findings none demands immediate code action in Phase 1, but **all** of them block clean Phase 3+ specification until the Phase 2 decision set below is made.

## 6. Cross-module dependency picture (verified at the seam, per 03/04)

| Seam | Status as verified |
|---|---|
| Front-office → HK (checkout DIRTY write, `check-out.handler.ts:470–481`; readiness gate consumes HK-written states, `room.aggregate.ts:30`) | Real and load-bearing on the *write* side; the reverse (HK status drives FO check-in) is the intended product flow per the aggregate — and it works only when inspection approval succeeds (silent-swallow risk). Legacy FO side has an `assignment` path that ignores housekeeping status entirely (empty `BLOCKED_HK_STATUSES`, `room-assignment.service.ts:55`). |
| Reservations room-move (reschedules against `housekeeping_status` CLEAN/INSPECTED, `scheduled-room-moves.spec.ts` gate; write contract F-25 real-Postgres-tested) | The *only* HK-adjacent write under regression protection. |
| Room service (guest requests reaching UI both ways via string protocol; `/room-service/requests*`) | Dual implementation with its own corrupted-history side effect (HK-030) |
| Lost-found | UI presents, backend missing claim endpoint (HK-001); traces verified dead-table-redirected to `housekeeping_traces` by FO d10 tests |
| Employees/HR | Staff join broken across unrelated id spaces (HK-012); no provisioning write path (HK-024) |
| Command-center widgets | HK widget provider exists; not load-bearing in current findings |
| Inventory/supply | Two *different* supply-request stacks exist; HK's is backend-orphaned with a dead approval chain (HK-037, Annex A) |

## 7. Verification posture (counts, and what that means)

- **0 module tests.** Every behavior claim in the register is source-verified, none runtime-verified.
- **16 peripheral tests** (12 API + 4 web) reference HK semantics; their aggregate blind-spot list matches the register's [D]/[R] top rows exactly (05 §3 matrix).
- **Best-pinned**: check-in-side readiness gate + `housekeeping_status` read-precedence; reservations room-move write contract on real Postgres.
- **Unpinned**: entire HK module; checkout DIRTY write (spec file has zero `DIRTY`/`housekeeping`/`room_status` matches); inspection approve/reject; all frontend HK UI.
- **Runtime execution**: none performed, by mandate; every "OBSERVED BEHAVIOR" claim must be treated as code-path-proven, not DB/HTTP-proven (05 §4). Transfer to runtime (or waiver) is a Phase-2 entry precondition (06 §6).

## 8. Unresolved questions (register Q/E + open decision set)

Restated fully in 06 §4. The eight item clusters that Phase 2's business-rules decision must resolve:

1. Inspection-approve-only re-entry policy and CLEAN/OOO/OOS ownership (HK-006/HK-014/HK-020).
2. `front_office_status` — owner or tombstone (HK-015).
3. Business cadence & "daily" semantics incl. `business_date`, shifts, workload credit (HK-004/009/031/036).
4. Discrepancy doctrine incl. the ratified missing table (HK-003) — resurrect or retire.
5. Purge-or-build for ~40 dead endpoints + 6 dead components (HK-001/HK-023) — deletion is pre-approved for backend dead files only.
6. Supply-request intended flow before any consumer exists (HK-037).
7. Assignment data model: `rooms`-JSON attendants vs `hk_room_assignments` (HK-011).
8. Legacy `room_status` mirror: retire or keep as write-through (HK-013).

## 9. Phase 1 exit assessment pointer

Full gate evaluation: **06_PHASE1_EXIT_ASSESSMENT.md** — **PASS** with one recorded caveat (runtime not executed). Per the audit mandate, no Phase 2 work is started by this document set.

## 10. Reader map

| Question | Where |
|---|---|
| How bad is each finding, exactly, and where is it in code | 02 (register) |
| What workflows actually run today | 03 (workflow maps, Mermaid) |
| What data models/tables/state fields exist, and who writes what | 04 (domain & data map); 05 §7 (writer census) |
| Was this verified — how, how far, what remains | 05 |
| Decided state of Phase-1 exit | 06 |
| Audit governance: scope/boundaries/lineage | 00 |

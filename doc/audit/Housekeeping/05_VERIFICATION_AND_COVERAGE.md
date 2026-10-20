# XYLO HOUSEKEEPING — VERIFICATION AND COVERAGE REPORT (DELIVERABLE 5)

Document: 05_VERIFICATION_AND_COVERAGE.md
Lifecycle step: Phase 1 — Forensic Audit
Status: COMPLETE
Date: 2026-10-20
Evidence basis: working tree at commit `ed278bc3` + uncommitted state (see 00_AUDIT_CONTROL.md §1).

---

## 1. Verdict in one paragraph

The Housekeeping module has **zero tests of its own**. Byte-for-byte, not a single test file exists under `apps/api/src/modules/housekeeping/` (no `__tests__` directory, no `*.spec.ts`), and `apps/web` has no test that imports the housekeeping store or any `_components` file. Sixteen peripheral tests (12 API + 4 web) reference housekeeping *concepts*, and their aggregate blind spots align precisely with the register's highest-severity findings: nothing pins `generateDailyTasks`, inspection approve/reject, task-status transitions, checkout→DIRTY, permission/property scoping, or any of the ~40 dead frontend endpoints. Runtime verification was not performed at all (§4 explains why and what that costs). One additional defect (HK-037) surfaced during verification cross-checking and was appended to the register.

---

## 2. Test inventory (complete, as found in the working tree)

### 2.1 Module-internal tests: 0 (VERIFIED FACT)

- `apps/api/src/modules/housekeeping/` contains no `__tests__/` directory and no `*.spec.ts` (verified by recursive glob); its 8+ files (controller, `housekeeping.service.ts`, inventory/stock/supply-request services, CQRS `application/` + `infrastructure/` + `domain/` + `events/`) are entirely uninstrumented.
- `apps/web/store/housekeepingStore.ts` (1,523 lines) and the twelve `apps/web/app/(dashboard)/housekeeping/_components/*.tsx` have no referencing test anywhere under `apps/web` (grep for `housekeeping` across `*.test.ts(x)` matched only unrelated FO/calendar fixtures of `housekeepingStatus` values, §2.2 web rows).
- `apps/web/features/housekeeping/` (the only React Query feature for this domain) — no tests.

### 2.2 Peripheral tests that touch housekeeping semantics (16)

API (12):

| # | Test file | What it actually pins | HK-relevant blind spot |
|---|---|---|---|
| 1 | `apps/api/src/modules/front-office/check-in/__tests__/room-readiness.spec.ts` | Readiness gate regressions: `READY` (schema default) assignable (:43–44); `null`/`undefined`/`PICKUP` not assignable (:48–53); `OUT_OF_SERVICE` blocks even when `housekeeping_status` reads CLEAN (:56–57); OCCUPIED ineligible (:66); **reading precedence — checkout-written `housekeeping_status` wins over `front_office_status`** (:85–92); fallback only when housekeeping absent (:112–128); input fabrication forbidden (`data).not.toHaveProperty('housekeeping_status')` :153) | Does not and cannot test that checkout actually *writes* DIRTY (unpinned upstream, row 12) |
| 2 | `…/check-in/__tests__/room-aggregate.spec.ts` | `isAssignableForCheckIn` for CLEAN/INSPECTED/READY vs DIRTY (:21–25); aggregate transition preserves INSPECTED (:71–75) | — |
| 3 | `…/check-in/__tests__/room-assignable.spec.ts` | Assignment evaluation rejects DIRTY room (:42) | — |
| 4 | `…/check-in/__tests__/application/queries.handler.spec.ts` | Ready-rooms query with a DIRTY-seeded room excluded (:170) | Source rows are mocked; no DB round-trip for the gate |
| 5 | `…/check-in/__tests__/application/assign-room.handler.spec.ts` | Assign with DIRTY room (:137) | Same mocking caveat |
| 6 | `apps/api/src/modules/front-office/__tests__/d10-dead-table-redirect.spec.ts` | Traces inventory reads/writes/auto-delete from `housekeeping_traces` only (:43, :71, :294–303, :333–360) | The legacy HK traces duplication is asserted *dead* — but no HK module test corroborates |
| 7 | `apps/api/src/modules/front-office/__tests__/d03-reservation-id-length.spec.ts` | ID-length sweep includes `housekeeping_traces` (:71) | Sweep only; no behavior |
| 8 | `apps/api/src/modules/front-office/application/queries/__tests__/get-room-detail.query.handler.spec.ts` | Room detail read carries `housekeeping_status` and OOO variant (:18, :220, :240) | Read only |
| 9 | `apps/api/src/modules/reservations/__tests__/scheduled-room-moves.spec.ts` | Room-move target gate: `housekeeping_status` CLEAN accepted (:86, :232), DIRTY rejected (:241–245) | Mocked `$queryRawUnsafe`; the same gate as FO `room-assignment.service.ts:55` empty `BLOCKED_HK_STATUSES` (see 02 register HK-016) is *not* what is exercised |
| 10 | `apps/api/src/modules/reservations/infrastructure/__tests__/reservation-schema-contract.postgres.spec.ts` | F-25: real-Postgres contract for rooms `room_status`/`housekeeping_status` writes in room-moves incl. audit log (:117, :340–365) | Only writer under test in the *room-status family*; HK-equivalent variants (ROOM MOVE) untested |
| 11 | `apps/api/src/modules/group-allotment/infrastructure/__tests__/delete-soft-a1-contribution.postgres.spec.ts` | Rooms DDL replica incl. defaults `front_office_status='CLEAN'`, `housekeeping_status='READY'` (:83) |DDL sanity only |
| 12 | `apps/api/src/modules/front-office/application/commands/check-out/__tests__/check-out.handler.spec.ts` | Checkout business flow | **Zero** matches for `DIRTY`, `housekeeping`, or `room_status` in the whole file (grep-verified 2026-10-20) → the DIRTY condition write at `check-out.handler.ts:470–481` is **completely unpinned** (register HK-006's hand-off depends on it) |

Web (4) — fixtures only; they exercise FO worklist/rack/calendar rendering and merely carry `housekeepingStatus` values in their room shapes:

| # | Test file | Note |
|---|---|---|
| 13 | `apps/web/features/front-office/components/dashboard/__tests__/phase4-worklists.test.tsx` | `housekeepingStatus:'READY'` fixture only (:132) |
| 14 | `apps/web/features/front-office/utils/__tests__/worklist-helpers.test.ts` | READY/DIRTY fixtures in generic worklist logic (:133–170) |
| 15 | `apps/web/features/front-office/components/rack/__tests__/room-detail-modal.test.tsx` | Asserts housekeeping status rendered distinct (:6, :82–102, :161) — the room-rack parity reference; nothing equivalent exists for the HK module itself |
| 16 | `apps/web/features/reservations/calendar/components/__tests__/reservation-calendar-page.test.tsx` | Calendar fixtures incl. `OUT_OF_ORDER` (:81–84) |

Well-pinned by these tests (the *only* housekeeping-adjacent behavior that is protected by regression net): the **check-in-side readiness gate** and the semantic precedence of `rooms.housekeeping_status` over `front_office_status` — plus the reservations room-move write contract on real Postgres.

---

## 3. Coverage gaps (what no existing test can catch)

Ordered by register relevance (IDs refer to 02_FINDINGS_REGISTER.md):

| Gap | Uncovered production behavior | Findings exposed |
|---|---|---|
| Daily generation | `generateDailyTasks` reset-all + skip-if-exists + unique-key mid-loop abort (`housekeeping.service.ts:162–207`) | HK-004, HK-009 |
| Inspection approve/reject | rooms + `room_status` mirror write, both individually `.catch(() => {})`, client-supplied inspector identity (`housekeeping.service.ts:560–680`) | HK-006, HK-008 |
| Task transitions | PATCH status endpoint accepts any→any (no matrix enforcement) | HK-026 |
| Bulk discrepancy resolve | Reported counts vs actual mutations | HK-010 |
| Staff join | `employees.employee_id`⋈`housekeeping_attendants.employee_id` mismatch (needs DB FK evidence) | HK-012 |
| Checkout handoff | DIRTY write at `check-out.handler.ts:470–481` (spec has zero references) | HK-006 (hand-off leg only already indirectly guarded by row-1 tests from the *read* side) |
| Authorization | 62 routes × 0 role/permission gates; property-scope generalities (HK module runtime property separation) | HK-007 |
| Supply requests | Entire lifecycle; verify late HK-037 approval dead-end behavior at runtime | HK-037 |
| Inventory/stock/laundry services | No tests at all; frontend writes to dead endpoints | HK-001, HK-002 |
| Person-discrepancy CQRS stack | No tests; feature unmouted; payload mismatch untestable except at runtime | HK-003 |
| Web HK store + components | No tests; dead components incl. crash-on-render `MinibarSection` | HK-022, HK-023 |

---

## 4. Runtime verification status (no execution was performed)

**Status: not executed — by scope decision, recorded here as a limitation.**

- No dev server, no `docker compose up`, no reverse proxy, no DB connection string, no authenticated HTTP request was issued by this audit.
- Constraints: Phase 1 mandate is read-only; standing repo convention for real-DB suites reserves `AVAILABILITY_TEST_DATABASE_URL` + database `xylo_test` (never `xylo_cloud`); bringing compose infra up to answer Phase-1 questions was judged disproportionate and environment-modifying.
- Cost: items listed as **OBSERVED BEHAVIOR** (e.g., schema table-presence claims — HK-003's `person_discrepancies` missing-table verdict rests on `packages/db/migration-baseline.declarations.json:89–99`, which is ratified rather than executed) remain code-path-proven but not runtime-proven. Nothing in this audit was downgraded for that, but every affected claim is labeled.
- Repayment site: Phase-2 Business Rules Decision entry or readiness review (06 §6/§7 lists the specific runtime checks to convert OBSERVED→VERIFIED, e.g., `HK-012` join affinity against a live `employees.id` FK and the `hk_sla_config` write-failure tolerance path).

---

## 5. Evidence-collection methods used (for reproducibility)

- Repo state: `git rev-parse HEAD`, `git log --oneline`, `git status --porcelain` (repo-wide and scope-filtered).
- Search: `grep` for endpoint paths, status literals, table names with repo-relative paths + line numbers; glob for test files and component files.
- Reading: full-file reads of the 12 most load-bearing HK sources; targeted line reads where a claim rested on specific lines (reported line counts were re-verified with PowerShell `[System.IO.File]::ReadAllLines(...).Length` counts after a `Measure-Object -Line` undercount was detected).
- Cross-verification: register claims re-checked against three evidence classes — source inspection, executed-audit lineage (`docs/reservations/phase-3/01_R8_CLASSIFICATION_LEDGER.md`, `packages/db/migration-baseline.declarations.json`), and render/import graph of web components.
- HK-037 late addition: single-file closed-system verification of the supply-request service transition map + all `PENDING_APPROVAL` literal sites (all 4 sites are map/counters, no writer).

## 6. Confidence-classification of Phase-1 claims

| Confidence tier | What it applies to |
|---|---|
| HIGH — VERIFIED FACT | File/line claims, route counts, endpoint cross-walk (Annex A of register), schema model contents, migration-baseline declarations, control record, all 12 mapped workflows' source paths |
| HIGH (single-file closed) | HK-037 (transition map is complete and self-guarding; no writer for PENDING_APPROVAL exists anywhere in the API) |
| MEDIUM — OBSERVED BEHAVIOR | Test *execution* semantics of the 16 peripheral tests; `(dashboard)`-route runtime render behavior of dead components; exact request-shape behavior under live HTTP |
| MEDIUM (source read, DB-dependent) | HK-012 join affinity, HK-009 collision freq, HK-003 table absence (ratified but not executed) |
| OPEN — UNVERIFIED | Items in register §E/[Q] + Phase-2 decision register (00 §5, 06 §4) |

## 7. Housekeeping-relevant writers of room condition — cross-check recap

During verification the full writer census for `rooms.housekeeping_status` / `room_status` / `front_office_status` was re-derived and re-matched to the register:

| Writer | Writes | Module | Pinned by tests? |
|---|---|---|---|
| Checkout handler condition write | rooms.`housekeeping_status`='DIRTY' (conditioned on reservation state) | `front-office` | **NO** (row 12) |
| HK inspection approve | rooms + legacy `room_status` mirror to INSPECTED, `.catch(()=>{})` swallowed | `housekeeping` | **NO** |
| Reservations room-move | rooms `room_status`/`housekeeping_status` + audit (F-25) | `reservations` | **YES** (row 10, real Postgres) |
| (No writer) | CLEAN, OOO, OOS, `front_office_status` | — | N/A (register HK-014/015/019/020) |

This table is the *only* writer census for room condition in the API as of the audited tree; it is the verification record behind register findings HK-006/HK-013–HK-016 and the FO-view of HK-019/HK-020.

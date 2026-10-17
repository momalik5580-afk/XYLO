# XYLO Front Office — Phase 1 Baseline & Exit Gate

- **Date:** 2026-10-16
- **Phase:** Phase 1 — Current-State Audit & Baseline (audit-only)
- **Companion documents:** `docs/front-office/PHASE_1_CURRENT_STATE_AUDIT.md`, `docs/front-office/PHASE_1_GAP_REGISTER.md`

---

## 1. Commands Executed and Actual Results

All commands were read-only or test-only. No installs, migrations, schema changes, or configuration changes were performed. Host is memory-constrained (~4 GB), so **whole-repo `typecheck`/`build` were intentionally not run** to avoid exhausting the machine.

| # | Command (working dir) | Purpose | Result |
|---|---|---|---|
| 1 | `npx jest "features/front-office"` (`apps/web`) | Run web FO unit/component tests | **PASS** — `Test Suites: 2 passed, 2 total`; `Tests: 8 passed, 8 total`; ~157 s. Includes `features/front-office/utils/__tests__/tax-computation.test.ts` and `features/front-office/components/dialogs/__tests__/fo-change-rate-failure.test.tsx`. Warning: "a worker process has failed to exit gracefully" (open-handle/teardown warning, non-failing). |
| 2 | `npx jest "modules/front-office" --silent` (`apps/api`) | Run backend FO unit tests | **BLOCKED — heap OOM / environment.** Jest spawned 7 parallel workers; the machine (~4 GB) exhausted the JS heap. One spec passed before the crash (`__tests__/front-office-events.spec.ts`, ~497 s), then `FATAL ERROR: Reached heap limit Allocation failed`. `__tests__/command-ownership.spec.ts` was terminated before running. Full output: see §5. |
| 3 | Read-only source inspection (`read`/`grep`/`glob`) | Verify routes, decorators, wiring, tests | Completed — evidence cited in the audit and register. |
| 4 | `git status --short` | Confirm no prohibited changes | See §6. |

### 1.1 Commands deliberately NOT run

| Command | Reason |
|---|---|
| `pnpm typecheck` / `turbo run typecheck` | Whole-repo `tsc --noEmit` is memory-heavy; host constrained to ~4 GB. |
| `pnpm build` / `turbo run build` | Same; heavy Next/Nest builds. |
| `pnpm test` (root/turbo) | Would run all workspaces' suites; scoped runs used instead. |
| `pnpm db:*`, any migration | Prohibited by task rules; no DB mutations allowed. |
| `npx playwright test` | Requires running app servers; not appropriate for audit-only, resource-constrained run. |
| API `lint` | No-op by design (`apps/api/package.json:12` → `echo 'ok'`). |

---

## 2. Tests Passed / Failed / Skipped / Blocked

### 2.1 Passed

- **Web FO suite (executed):** 2 suites, 8 tests — see §1 row 1.
- **Backend FO suite:** recorded in §5.

### 2.2 Failed

- **Backend FO run crashed on heap OOM** (environment), not on a test assertion. `__tests__/front-office-events.spec.ts` PASSED; the worker was then OOM-killed and `__tests__/command-ownership.spec.ts` reported "Test suite failed to run … terminated by another process: signal=SIGTERM". No test assertion was observed to fail before the crash.

### 2.3 Skipped (by design / environment)

- `apps/api/src/modules/front-office/check-in/__tests__/check-in-availability.postgres.spec.ts` is gated by `describePostgres` (`apps/api/src/modules/reservations/infrastructure/__tests__/availability-postgres.harness.ts:8-9`), which resolves to `describe.skip` unless `AVAILABILITY_TEST_DATABASE_URL` is set. Without that env var the test is **skipped**. (Not confirmed to be set in this environment.)
- No `.skip`/`xit`/`todo` markers were found in FO test sources.

### 2.4 Blocked / not executed

- **Backend FO unit suite — BLOCKED:** crashed with a Node heap out-of-memory error on the ~4 GB host. Only `front-office-events.spec.ts` completed and passed; the remainder of the suite was not executed to completion. This is a **baseline limitation / environment block**, not a pass. Re-run on a host with more memory (or with `--maxWorkers=1` and a raised heap) before relying on it.
- Whole-repo typecheck/build — blocked by host memory constraint (see §1.1). This is a **baseline limitation**, not a pass.
- Playwright E2E — not executed (requires live servers).

---

## 3. Backend FO Test Inventory (reference)

27 test files exist under `apps/api/src/modules/front-office/**` (check-in stages, check-out, transfer, upgrade, change-rate, route permissions, command ownership, events, D-03/D-10 remediations, room detail). Uncovered capabilities are listed in `FO-P1-012`.

---

## 4. Baseline Limitations

1. **Backend FO unit suite could not complete** on the ~4 GB host: Jest spawned 7 parallel workers and the process crashed with `FATAL ERROR: Reached heap limit Allocation failed`. Only one spec passed before the crash. The backend FO unit baseline is therefore **not verified** by this phase.
2. **Typecheck/build not run** on the full monorepo due to the same memory constraint. Frontend/backend compile status is **unverified** by this phase.
3. **No runtime/API calls** — all guard-enforcement, validation, and skip behavior findings are derived from source reading.
4. **Working tree is heavily modified** (large uncommitted diff: many deletions/modifications across `apps/*`, `packages/*`, `docs/*`). The audited state is the **working tree**, not a clean commit (`HEAD = ed278bc`). Findings describe the working tree.
5. `apps/mobile` was not scanned.
6. API `lint` provides no signal (no-op script).

---

## 5. Backend FO Scoped Test Run — Actual Result

> Command: `npx jest "modules/front-office" --silent` run from `apps/api` (Jest config inline in `apps/api/package.json`, `rootDir: src`, `testRegex: .*\.spec\.ts$`, ts-jest `diagnostics:false`).

**Actual result (verbatim excerpts):**

```
PASS src/modules/front-office/__tests__/front-office-events.spec.ts (497.396 s)
...
<--- Last few GCs --->
[22024:...] 3035544 ms: Scavenge (during sweeping) 4021.9 (4111.0) -> 4017.2 (4127.8) MB ...
[22024:...] 3673490 ms: Mark-Compact (reduce) 4071.3 (4160.4) -> 4034.6 (4100.2) MB ...
FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
----- Native stack trace -----
FAIL src/modules/front-office/__tests__/command-ownership.spec.ts
  ● Test suite failed to run
    A jest worker process (pid=22024) was terminated by another process: signal=SIGTERM ...
Exited with code 1
```

**Interpretation:**

- Jest defaulted to 7 parallel workers (one per logical CPU). Combined with the running `pnpm dev` servers, the process set exceeded the host's ~4 GB and exhausted the V8 heap.
- **Passed before crash:** 1 spec — `__tests__/front-office-events.spec.ts`.
- **Terminated by the OOM kill (not a test-assertion failure):** `__tests__/command-ownership.spec.ts`.
- **Not executed to completion:** the remaining ~25 FO specs.

**Conclusion:** the backend FO unit baseline is **BLOCKED / NOT VERIFIED** in this environment. It was **not** observed to fail on any assertion. To obtain a valid result, re-run on a host with more RAM, or with `node --max-old-space-size=<higher> ./node_modules/.bin/jest "modules/front-office" --runInBand` (single worker). Do not treat this as a pass.

> Note: this audit did not re-run the suite after the OOM, to avoid further exhausting the constrained host.

---

## 6. Prohibited-Change Check

- No application code, schema, migration, configuration, or dependency files were modified.
- The only files created are the three Phase 1 deliverables under `docs/front-office/`.
- No commits, pushes, deploys, or environment changes were made.
- `git status` continues to show the pre-existing dirty tree (unrelated to this phase); the new `docs/front-office/` files appear as untracked additions.

**Verification method:** `git status --short` before and after; the new files are additive and confined to `docs/front-office/`.

---

## 7. Phase 1 Exit Criteria and Evidence

| # | Exit criterion | Status | Evidence |
|---|---|---|---|
| 1 | FO routes/pages/features inventoried to a defensible level | **MET** | Audit §4; 54 controller routes + 11 check-in routes; 7 tabs; 13 dialog kinds; feature slice tree. |
| 2 | Frontend↔backend dependencies traced or marked unverified | **MET** | Audit §5 dependency map; `apps/mobile` marked UNKNOWN. |
| 3 | Existing test coverage and baseline results documented accurately | **MET (with limitations)** | Baseline §1–§5; web FO suite executed and passed; backend FO suite blocked by heap OOM and recorded as NOT VERIFIED; typecheck/build explicitly not run. |
| 4 | UI/UX assessment distinguishes verified defects from recommendations | **MET** | Audit §8; FO-P1-003/004/007/008/009/010/014/015. |
| 5 | Gap register has evidence and avoids duplicates | **MET** | Register: 17 unique findings, each with file:line evidence. |
| 6 | Reservations artifacts required by Phase 2 identified without reopening work | **MET** | Audit §11 lists specs, roadmap, forensic baseline, backend/frontend integration points. |
| 7 | Audit documentation reviewed for contradictions/unsupported claims | **MET** | Finding IDs cross-referenced; severity counts reconciled (P1×1, P2×6, P3×7, Info×3); claims marked by evidence class. |
| 8 | No prohibited implementation/schema/config/deploy changes | **MET** | §6. |

**Exit declaration:** Phase 1 is **COMPLETE**, with two recorded environment limitations: (1) the backend FO unit suite could not complete on the ~4 GB host (heap OOM; one spec passed, no assertion failure observed) and is recorded as **NOT VERIFIED**; and (2) the full-monorepo `typecheck`/`build` was not executed for the same reason. All audit content is grounded in the working-tree source. These limitations **do not change** the audit's conclusion — the findings are code-direct — but the backend FO test baseline must be re-verified on a less constrained host before or during Phase 2.

---

## 8. Proposed Phase 2 Investigation Inputs

Phase 2 — **Reservations Integration & Linkage Audit** — should consume the following and verify:

1. **Reservations→FO entry points:** `features/reservations/model/reservation-actions.ts` (`buildCheckInHref()`), reservations workspace, `OperationalPulseStrip`; confirm handoff parameters (`tab=arrivals&dialog=check-in&reservation=<id>`) are honored end-to-end.
2. **Shared backend writers:** confirm the single-writer contract (`IReservationRepository` methods `applyCheckIn`, `applyCheckOut`, `applyReverseCheckIn`, `assignRoomNumber`, `applyUpgrade`, `applyRateCodeChange`) is honored by all FO handlers (partially evidenced by `command-ownership.spec.ts`).
3. **Delegated commands:** FO `extend`/`reinstate` delegate to Reservations; verify status/state-machine consistency and change-logging.
4. **Availability integration:** verify the FO check-in/check-out availability-port usage (`check-in-availability.postgres.spec.ts`, `check-out.handler.ts`) against the availability module contract.
5. **Billing/folio linkage:** verify `CheckoutFlowModal` merges (`billingStore` + `features/cashiering` `useReservationFolios`) produce a single authoritative balance.
6. **Authorization cross-check:** reconcile FO permission codes (`front_office:*` vs `reservations:reservation:*`) and the missing-decorator gap (FO-P1-001).
7. **Reservations specs** listed in Audit §11 (do not re-audit completed Reservations work).

---

*End of Phase 1 baseline & exit gate.*

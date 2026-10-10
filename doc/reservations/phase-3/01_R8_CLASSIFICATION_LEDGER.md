# B3.5 — R8 Classification Ledger (RETAIN/DELETE, classify-only)

**Status:** LEDGER REMEDIATED — READY FOR OWNER REVIEW (correction pass 2026-10-10). **No deletion performed** (execution is B4.3 and requires authorization + zero-reference re-verification at deletion time).

**Authority (single source):** `docs/reservations/phase-1/02_PHASE1_CLOSURE_AND_ROADMAP.md`

- **R8 row** (findings table): dead/legacy candidates — `components/reservations/events/` (23 files), `activities/`, 2 modals, `ReservationToolbar`, 9 orphan config endpoints that would 404, duplicate nav; ARCH-042: 82 zero-importer files repo-wide (incl. Reservations Phase-6 port adapters). Status `SUSPECTED / classify-only`. Resolution: *Classify → RETAIN/DELETE ledger; deletion only in Phase 4 after zero-reference evidence + authorization. Tests: importer-count evidence per item at deletion time.* Batch split: classify **Phase 3 / B3.5**; delete **Phase 4 / B4.3**.
- **B3.5 row:** scope = *R8 classification (no deletion): RETAIN/DELETE ledger for dead components, orphan endpoints, ARCH-042 zero-importer list*; acceptance = *ledger drafted with importer-count evidence; owner review queued*; focused tests = *evidence recorded, not a test*; deps = B3.1.
- **Phase 3 exit gate:** *B3.5 ledger exists (deletion NOT performed).*
- **B4.3 row:** *Execute RETAIN/DELETE ledger (R8) with authorization; deletion only after zero-reference re-verification at deletion time.*

**Supporting sources:** `docs/reservations/phase-1/01_PHASE1_ASSESSMENT_REPORT.md` §2 item 18 (cited as "§2.18") and risk row R8; `docs/architecture/01_CURRENT_ARCHITECTURE_AUDIT.md` §10.1 item 4 + §13 ARCH-042 (audit method §1.2).

**Conflict/discrepancy check vs other approved sources:** `docs/enterprise/reservations-roadmap-v2.md` (tracking layer) contains **no B3.5 definition and no conflict** — its Phase 11 exit gate (*RETAIN/TRANSFORM/MERGE/ARCHIVE/DISCARD ledger **executed***) is execution-scope (B4.3/Phase 11), not classification. Minor source defects recorded, not blocking: (a) assessment has no `§2.18` heading — the cited content is §2 item 18 (verified); (b) R8's reported counts (`23` event files, `9` config endpoints, `82` zero-importer files) were `[R]`-marked reported values — current counts differ (see §3/§4/§5.1 drift notes).

---

## 0. Remediation log (correction pass 2026-10-10)

Applied after a full re-verification pass (re-runs of both scans + a whole-repo reference sweep + registration/endpoint re-proofs). All changes are rule-internal corrections; no owner decision was invented.

1. **Coupling contradictions fixed.** B2 (`DELETE`) ↔ C82 (`RETAIN`) and B4 (`DELETE`) ↔ C42 (`DELETE` while its FE client existed) violated the G/pair rules → B2, B4, C42 now `RETAIN` (G/pair); see §3.1/§4.1.
2. **Rule misapplications corrected.** C26/C27 were `RETAIN W` for a mechanism that does not reach them (§4 rows) → now `DELETE`; C31 matched G (unregistered handler + existing FE client chain) but was `DELETE U` → now `RETAIN` (G).
3. **Stale cross-references fixed:** B2 cited nonexistent "C3 row" and §3 extras cited "C3/C11 rows" → now C82 / C81+C83+C92 (§3.1).
4. **Q dispositions (§7):** Q2 and Q4 resolved by evidence; Q5/Q6/Q7 resolved at technical level with authorization folded into Q8; Q1/Q3 remain open owner rulings with complete evidence.
5. **Evidence added:** whole-repo reference sweep (§1.6), git-history dating attempt (impossible — §5.1), schema/DDL gap for the wiring-gap handlers (§4.1), FE triplet importer closure checks (§3/§3.1), prefix-chain endpoint proof (§3).

---

## 1. Method (evidence, not tests)

No product or test code touched. Evidence collected 2026-10-10 by read-only scans (re-run in the remediation pass; results identical):

1. **Zero-importer scan (backend, ARCH-042 class):** re-implementation of audit §1.2 method over `apps/api/src` — enumerate non-test `.ts` files (exclude `*.spec.ts`, `*.test.ts`, `__tests__/`, `*.d.ts`); extract every `from '…'`, `import '…'`, `import('…')`, `require('…')` specifier; resolve relative and `@/*` (→ `src/*`) specifiers to absolute paths (`+ .ts`, `/index.ts`); count **non-test** importers per file; list files with count 0. Script kept out of the repo (temp): `C:\Users\Pro\AppData\Local\Temp\opencode\zero-importer-scan.js`. Re-run this pass → identical totals (1320/110; named 8/8).
2. **Importer/render counts (frontend, R8 class):** `rg` for import specifiers and JSX render sites per candidate path/symbol across `apps/web`. This pass additionally closed every FE deletion unit: each of the 9 config triplets + the 5 §3.1 triplets is imported only by its own panel/hook + the feature `index.ts` barrels → deletion units = triplet + dead export lines in `features/*/hooks/index.ts`, `features/*/api/index.ts` (and feature barrels where applicable).
3. **Endpoint existence (orphan-endpoint class):** each frontend endpoint literal matched against backend route strings repo-wide (`rg` over `apps/api/src`, all `*.controller.ts` and non-controller files — empty for every §3/§3.1 endpoint). **Prefix-chain proof:** FE fetch `API_BASE + path` (`apps/web/services/api.ts:1,81`, `API_BASE='/api'`) → `next.config.js` rewrite `/api/:path*` → `${API_URL}/:path*` with default `http://localhost:4000/api/v1` (`next.config.js:25,60-61`) → backend `app.setGlobalPrefix('api/v1')` (`apps/api/src/main.ts:21`). A route string absent anywhere under `apps/api/src` therefore 404s regardless of prefix.
4. **Registration checks:** module/provider references in `app.module.ts`, `worker.module.ts`, `*.module.ts`, and all `configure(consumer)`/`app.use` sites (CQRS handlers are registered by manual import in module files — AGENTS.md). Middleware registration exists only for `CorrelationIdMiddleware` (`common.module.ts:57-58`).
5. **False-positive sweep (original):** (a) string-path/`require.resolve`/config-path wiring check (`workflowsPath`); (b) spec-file string-reference sweep over all 110 zero-importer paths (`spec-string-pin-sweep.js`, temp) — hits listed in §5.3.
6. **Whole-repo reference sweep (new, this pass):** `full-repo-sweep.js` (temp) builds 433 fixed-string patterns from the 110 paths (full repo path, api-relative path, path-minus-extension, basename stem, barrel parent-dir) and runs one `rg -F` pass over **all file types** (source, docs, configs; excludes `node_modules/.git/.next/dist`, lockfiles, this ledger). Attribution filters: exact-path patterns counted as-is; barrel-dir patterns only when **not** followed by a sibling path segment (so `common/cqrs/command-bus` deep imports do not count as barrel references); basename stems only when followed by `.ts`; self-hits excluded; `PROJECT_STRUCTURE.md` tree listings tagged `[tree-doc]` (informational). Result: **219 kept hit-lines; 97/110 files have ≥1 reference outside this ledger; 13 files have zero references anywhere** (listed in §5.3.1). Report: `C:\Users\Pro\AppData\Local\Temp\opencode\full-repo-sweep-results.txt`.
7. **Git-history dating attempt (new, this pass):** impossible — repository has **5 commits**; representative zero-list files (e.g. `share-advanced.dto.ts`) and even the audit document (`docs/architecture/`) are **untracked** (`??`). File-add dating cannot attribute the 82→110 drift (§5.1).
8. **Schema/DDL cross-check (new, this pass):** wiring-gap handler SQL targets compared against `packages/db/schema.prisma` + all three migration trees (§4.1).

**Scan result:** 1320 non-test `.ts` files in `apps/api/src`; **110 files with zero non-test importers** (audit recorded 82 at audit time — drift note §5.1); of the 110, 4 have test-only importers (`partition-router.middleware` 1, `fx-currency.service` 2, `reservation-party.value-object` 1, `prisma-inventory-reservation.adapter` 1 — the adapter additionally has a cross-app `fs.existsSync` pin in `apps/web/features/availability/__tests__/t542-dead-artifacts.test.ts:97`); the other 106 have zero importers of any kind. Ledger path table ↔ scan list compared 110/110 identical (slash-direction only). Per the whole-repo sweep, **13 of the 110 have no reference of any kind anywhere in the repo outside this ledger** (§5.3.1).

**Named ARCH-042 members verified present in the current scan (8/8):** `prisma-front-office-handoff.adapter.ts`, `prisma-guest-profile.adapter.ts`, `prisma-inventory-reservation.adapter.ts`, `front-office-stay.aggregate.ts`, `tenant-injection.extension.ts`, and the 3 unwired reservations handlers (`get-l2b-config`, `get-email-templates`, `get-package-exclusions`).

**Classification rules used (applied uniformly; owner review may override):**

| Code | Rule | Verdict |
|---|---|---|
| E | NestJS entrypoint — zero importers by design (executed by runtime) | RETAIN |
| W | Wired by a non-import mechanism that **verifiably reaches the file** (`TEMPORAL_WORKFLOWS_PATH` default → `./dist/infrastructure/temporal/workflows` with `TEMPORAL_WORKER_ENABLED=true` (`.env:59`); t567 content-read pins). Mechanism claim must be checked per file — not inherited by directory | RETAIN |
| R | Roadmap deliverable (Reservations Phase-6 ports/aggregate) — R8 explicitly names the adapters | RETAIN |
| G | Wiring gap — backend handler exists but unregistered **and** a frontend API client chain exists | RETAIN (owner: wire or delete pair) |
| P | Pair consistency — a G pair (backend handler + FE chain) must share one verdict; the FE side of a G pair is RETAIN even if unmounted | RETAIN |
| M | Conflicts with architecture documentation — retain pending owner ruling | RETAIN |
| B | Barrel `index.ts` never imported; its content is live via deep imports (barrel deletion cannot affect live code) | DELETE |
| L | Audit-declared legacy residue, superseded by a live mechanism | DELETE |
| U | Unregistered / unreferenced dead code (no route, no registration, no consumer, no string-path reference) | DELETE |
| S | Unreferenced scaffold / DTO / value-object / duplicate service | DELETE |

Conservative default: any row with conflicting or incomplete evidence is RETAIN/M (cost of a missed deletion is zero now; deletions are re-verified + authorized at B4.3).

---

## 2. Class A — Dead frontend components (R8 enumerated items)

Importer/render evidence = current counts across `apps/web` (excluding the candidate's own definition lines).

| # | Item | Path | Files now | Non-test importers | Render sites | Verdict | Basis / evidence |
|---|---|---|---|---|---|---|---|
| A1 | Events suite | `apps/web/components/reservations/events/` | **18** | barrel `reservations/events` imported **0**×; sole external edge = `activities/ActivitiesTab.tsx:7` → `../events/FunctionDiary` | 0 outside the tree | **DELETE** | Unreachable: tree root `ActivitiesTab` itself has 0 importers (A2), so the single external edge never executes. Reported "23 files" vs current 18 = drift (audit-time count was `[R]`). Deletion unit = whole `events/` dir incl. `index.ts`. |
| A2 | Activities suite | `apps/web/components/reservations/activities/` | 2 | **0** | 0 | **DELETE** | `rg "from .+activities"` → no matches; `ActivityDashboard`/`ActivitiesTab` referenced only by each other. |
| A3 | Edit modal | `.../modals/EditReservationModal.tsx` | 1 | **0** | 0 (`<EditReservationModal` absent) | **DELETE** | No imports, no barrel (no `components/reservations/index.ts` exists), no render. |
| A4 | Detail modal | `.../modals/ReservationDetailModal.tsx` | 1 | **0** | 0 | **DELETE** | Same as A3. |
| A5 | Toolbar | `.../ReservationToolbar.tsx` | 1 | **0** | 0 | **DELETE** | Same as A3. |
| A6 | Duplicate nav (dead side) | `apps/web/features/reservations/workspace/reservation-workspace-nav.tsx` | 1 | exported only by `workspace/index.ts:3` and `features/reservations/index.ts:85` (barrels) | **0** (`<ReservationWorkspaceNav` absent) | **DELETE** | Exported-but-never-consumed; duplicate of A7. |
| A7 | Duplicate nav (live side) | `apps/web/features/reservations/workspace/reservation-sidebar-nav.tsx` | 1 | imported by `reservation-workspace-layout.tsx:8` | rendered `reservation-workspace-layout.tsx:34` | **RETAIN** | Live in the current workspace layout. R8 "duplicate nav" resolved: keep A7, ledger A6 for deletion. |

Note: A3–A6 deletion also removes their dead-export lines from `features/reservations/index.ts` / `workspace/index.ts` where applicable (partial-file units — re-verify at B4.3).

## 3. Class B — Orphan config endpoints (reported: "9 endpoints that would 404")

Evidence per row: frontend panel mount state (0 render sites unless noted) + backend route existence (repo-wide route-string grep + prefix-chain proof, §1.3). FE hook/api directories: `features/reservations` (B1–B3), `features/front-office` (B4), `features/housekeeping` (B5–B7), `features/settings` (B8–B9).

| # | Config panel (mount state) | Client files (hook → api) | Endpoint | Backend route evidence | Verdict |
|---|---|---|---|---|---|
| B1 | `ChangesLogConfigPanel` (0 renders) | `hooks/use-changes-log-config.ts` → `api/changes-log-config.api.ts` | `GET /reservations/changes-log-config` | **absent** (no `changes-log-config` string in any `apps/api/src` file) | **DELETE** (3 files) |
| B2 | `L2BConfigDisplay` (0 renders) | `hooks/use-l2b-config.ts` → `api/l2b-config.api.ts` | `GET /reservations/l2b-config` | **absent** — `GetL2BConfigHandler` exists but is **unregistered** (C82) | **RETAIN** (3 files) — P pair with C82; owner decides pair (Q3) |
| B3 | `AutoCancelSweepButton` (0 renders) | `hooks/use-auto-cancel.ts` → `api/auto-cancel.api.ts` | `GET /reservations/auto-cancel/config` | **absent** — only `POST auto-cancel/sweep` exists (`reservations.controller.ts:521`, live) | **DELETE** (3 files; sweep backend stays live/headless — no G pair, no config handler exists) |
| B4 | `TracesInventoryPanel` (0 renders) | `hooks/use-traces-inventory.ts` → `api/traces-inventory.api.ts` | `GET /front-office/traces-config`, `POST /front-office/traces/auto-delete`, `POST /front-office/traces/sync-inventory` | **all absent** (`modules/front-office/front-office.controller.ts` has live `traces` CRUD only, `:152-162`, served by `CrsFrontDeskIntegrationService`) — but `TracesInventoryService` implements exactly these three operations (C42) | **RETAIN** (3 files) — P pair with C42; owner decides pair (Q3) |
| B5 | `TurndownAttributesConfig` (0 renders) | `hooks/use-turndown-attributes.ts` → `api/turndown-attributes.api.ts` | `GET/PUT /housekeeping/turndown-attributes` | **absent**; no backend implementation (VO C54 only) | **DELETE** (3 files) |
| B6 | `CreditRulesConfig` (0 renders) | `hooks/use-credit-rules.ts` → `api/credit-rules.api.ts` | `GET/PUT /housekeeping/credit-rules` | **absent**; no backend implementation (VO C52 only) | **DELETE** (3 files) |
| B7 | `MembershipSchedulingConfig` (0 renders) | `hooks/use-membership-scheduling.ts` → `api/membership-scheduling.api.ts` | `GET/PUT /housekeeping/membership-scheduling` | **absent**; no backend implementation (VO C53 only) | **DELETE** (3 files) |
| B8 | `MembershipRateRuleConfig` (0 renders) | `hooks/use-membership-rate-rule.ts` → `api/membership-rate-rule.api.ts` | `GET/PUT /settings/membership-rate-rule` | **absent**; no backend implementation (VO C95 only) | **DELETE** (3 files) |
| B9 | `LoyaltyEnrollmentConfig` (0 renders) | `hooks/use-loyalty-enrollment.ts` → `api/loyalty-enrollment.api.ts` | `GET/PUT /settings/loyalty-enrollment` | **absent**; no backend implementation (VO C89 only) | **DELETE** (3 files) |

**Count reconciliation:** exactly **9** unmounted config panels whose primary config `GET` has no backend route — reproduces the reported "9 orphan config endpoints that would 404" (`[R]`) under the one-config-GET-per-panel reading. Route absence is unchanged by this pass; FE-side verdicts: **7 DELETE, 2 RETAIN** (B2/B4 held with their G-pairs under Q3). Auxiliary endpoints of the same panels (PUT verbs, `traces/auto-delete`, `traces/sync-inventory`) are in the same units.

**Verified live (excluded — do NOT delete):** `GET /front-office/tax-config` (`modules/front-office/front-office.controller.ts:146`, consumed by live `CheckoutFlowModal`); `GET/PUT /housekeeping/config/sla` (`housekeeping.controller.ts:236,241`); `GET/PUT /housekeeping/config/status-flow` (`:246,251`); `GET/PUT /pbx-config/:id` (`pbx-config.controller.ts:4`).

### 3.1 Additional orphan-endpoint pairs (outside the reported 9 — classified this pass)

FE units verified closed (panel/hook/api import only each other + feature barrels). Endpoints verified absent repo-wide (§1.3).

| # | FE unit (0 renders) | Files | Endpoint | Backend counterpart | Verdict |
|---|---|---|---|---|---|
| X1 | Email templates list | `features/reservations/components/EmailTemplatesList.tsx` → `api/email-templates.api.ts` | `GET /reservations/email-templates` | `GetEmailTemplatesHandler` **exists, unregistered** (C81) | **RETAIN** — P pair with C81 (Q3) |
| X2 | Package exclusions list | `features/reservations/components/PackageExclusionsList.tsx` → `api/package-exclusions.api.ts` | `GET /reservations/package-exclusions` | `GetPackageExclusionsHandler` **exists, unregistered** (C83) | **RETAIN** — P pair with C83 (Q3) |
| X3 | L2B feedback list | `components/L2BFeedbackMessagesList.tsx` + `hooks/use-l2b-feedback-messages.ts` → `api/l2b-feedback-messages.api.ts` | `GET /reservations/l2b-feedback-messages` | **none — no handler exists even unregistered** (repo-wide grep empty); VO C92 is orphaned modeling | **DELETE** (3 files; VO C92 joins unit — no pair, nothing to wire) |
| X4 | Batch CC auth | `components/BatchCCAuthButton.tsx` + `hooks/use-batch-cc.ts` → `api/batch-cc.api.ts` | `POST /cashiering/batch-cc-authorization` | `BatchCCAuthorizationHandler` **exists, unregistered** (C31); `cashiering.controller.ts` has no such route | **RETAIN** — P pair with C31 (Q3). Origin: `docs/Antigravity/v2/implementation_plan.md` P2-5 (unchecked boxes; unreferenced by any authoritative source) |
| X5 | Property description | `features/settings/components/PropertyDescriptionForm.tsx` + `hooks/use-property-description.ts` → `api/property-description.api.ts` | `GET/PUT /settings/property-description` | **none** (repo-wide grep empty) | **DELETE** (partial-file unit — the hook also exports `settingsKeys`/`getPropertyId`/`withProperty` used by B8/B9 hooks `use-membership-rate-rule.ts:5`, `use-loyalty-enrollment.ts:5`; deletion order matters at B4.3) |
| X6 | room-configs store actions | `apps/web/store/housekeepingStore.ts:1355-1388` (4 methods, not a triplet) | `GET/POST/PUT/DELETE /housekeeping/linen/room-configs` | **none** | **DELETE-candidate** — store methods have 0 UI callers; partial-file unit (owner note at B4.3) |

## 4. Class C — ARCH-042 zero-importer list (backend, current scan)

**Totals:** 1320 non-test files scanned; **110 zero-non-test-importer** (audit-time: 82 → +28 drift, §5.1). Verdicts: **17 RETAIN, 93 DELETE.** *Membership revised this pass (totals unchanged): C26/C27 RETAIN→DELETE (W misapplied — mechanism does not reach them), C31/C42 DELETE→RETAIN (G applies).* Non-test importers = 0 for every row by construction; test importers shown.

| # | Path (`apps/api/src/…`) | Test | V | Basis |
|---|---|---|---|---|
| 1 | `common/audit/audit.interceptor.ts` | 0 | DELETE | U (tree-doc listing only) |
| 2 | `common/authorization/index.ts` | 0 | DELETE | B (content live via deep import; dir references in audit docs are to the live module, not the barrel) |
| 3 | `common/base/index.ts` | 0 | DELETE | B (zero refs of any kind — sweep clean) |
| 4 | `common/cqrs/index.ts` | 0 | DELETE | B (e.g. `cashiering.service.ts:2` deep-imports `common/cqrs/command-bus`; doc hits are dir references to the live CQRS package) |
| 5 | `common/database/repositories/index.ts` | 0 | DELETE | B (audit ARCH-014: shared repository layer itself barely used; barrel not imported) |
| 6 | `common/events/base-event.ts` | 0 | DELETE | U (tree-doc only) |
| 7 | `common/events/event-handler.interface.ts` | 0 | DELETE | U (tree-doc only) |
| 8 | `common/helpers/async-helpers.ts` | 0 | DELETE | U |
| 9 | `common/helpers/crypto.helper.ts` | 0 | DELETE | U |
| 10 | `common/outbox/index.ts` | 0 | DELETE | B (content live: `common.module.ts:17`, `event.module.ts:6-7`; doc hits are `common/outbox/*` dir refs to live files) |
| 11 | `common/queue/worker-base.ts` | 0 | **RETAIN** | W — **pin-only**: t567 content-read (`t567-async-scoping.spec.ts:49,58,356`). Runtime-dead per audit:584 ("never extended") + `02_OPEN_EVIDENCE_RESOLUTION.md:1141` ("never called; live path is `@Processor` only") → W is test-pin, not runtime wiring; B4.3 may delete together with pin update (owner note, Q8) |
| 12 | `common/response/pagination.dto.ts` | 0 | DELETE | S |
| 13 | `common/response/response.types.ts` | 0 | DELETE | S |
| 14 | `common/storage/storage.types.ts` | 0 | DELETE | S |
| 15 | `common/swagger/swagger.constants.ts` | 0 | DELETE | U |
| 16 | `common/testing/fixtures.ts` | 0 | DELETE | U (generic-name doc hit `phase5-implementation-governance.md:42` is a filename convention, not a reference) |
| 17 | `common/testing/testing.module.ts` | 0 | DELETE | U |
| 18 | `common/validation/cross-field.validator.ts` | 0 | DELETE | U |
| 19 | `common/validation/unique.validator.ts` | 0 | DELETE | U |
| 20 | `core/filters/http-exception.filter.ts` | 0 | DELETE | U — audit:254 confirms `APP_FILTER` is `common/exceptions/exception.filter.ts`; same class name, 0 importers |
| 21 | `core/guards/compliance.guard.ts` | 0 | DELETE | U |
| 22 | `core/guards/partition-access.guard.ts` | 0 | DELETE | U |
| 23 | `core/interceptors/audit.interceptor.ts` | 0 | DELETE | U |
| 24 | `core/middleware/partition-router.middleware.ts` | 1 | **RETAIN** | M — unregistered (only `t564` imports it; registration search: `main.ts`, `app.module.ts`, all `configure()` — only `CorrelationIdMiddleware` at `common.module.ts:57-58`). Conflicts: `AGENTS.md:48` claims it active; **but** `availability-phase3-reservation-lifecycle-forensic-audit.md:713` already declares it dead; audit:318/ARCH-008 same → owner ruling Q1 |
| 25 | `infrastructure/prisma/extensions/tenant-injection.extension.ts` | 0 | DELETE | L — audit:318 "never called" + `02_OPEN_EVIDENCE_RESOLUTION.md:199` "0 importers"; superseded by ALS/`property-scope.guard` path (docs agree with deletion) |
| 26 | `infrastructure/temporal/activities/index.ts` | 0 | DELETE | U — content is `export {}`; **not** wired by any mechanism: `TEMPORAL_WORKFLOWS_PATH` loads `.../temporal/workflows` (not `/activities`), worker receives activities via `registerActivities()` (`temporal-worker.service.ts:21-23`), sweep found zero string refs (§1.6 clean). *Was RETAIN W — W misapplied; corrected this pass* |
| 27 | `infrastructure/temporal/index.ts` | 0 | DELETE | B — barrel re-exports `temporal.module` etc.; consumers deep-import (`app.module.ts:53`, `worker.module.ts:3`); `workflowsPath` loads the `workflows/` subdir, not this file. *Was RETAIN W — W misapplied; corrected this pass* |
| 28 | `infrastructure/temporal/workflows/index.ts` | 0 | **RETAIN** | W — content `export {}` but it **is** the string load target: default `./dist/infrastructure/temporal/workflows` (`temporal-config.service.ts:41`, env unset) with `TEMPORAL_WORKER_ENABLED=true` (`.env:59`); worker resolves it (`temporal-worker.service.ts:118-127`). Deletion breaks worker bootstrap |
| 29 | `main-worker.ts` | 0 | **RETAIN** | E (entrypoint; pinned by `ws-n-scan-gates.spec.ts:332,334`; comment ref `temporal-worker.service.ts:38`) |
| 30 | `main.ts` | 0 | **RETAIN** | E (entrypoint; many doc refs to bootstrap; t521 comment pin) |
| 31 | `modules/cashiering/application/commands/batch-cc-authorization/batch-cc-authorization.handler.ts` | 0 | **RETAIN** | G — FE client chain exists and is closed (`batch-cc.api.ts` + `use-batch-cc.ts` + `BatchCCAuthButton.tsx`, §3.1 X4); no route (`cashiering.controller.ts` route list enumerated). Pair with X4 → Q3. *Was DELETE U — G applies; corrected this pass* |
| 32 | `modules/cashiering/application/commands/create-daily-routing/create-daily-routing.handler.ts` | 0 | DELETE | U — no route, **no FE reference at all** (G's FE-side condition fails) |
| 33 | `modules/cashiering/application/queries/get-routing-for-day/get-routing-for-day.handler.ts` | 0 | DELETE | U (same) |
| 34 | `modules/cashiering/domain/errors/cashiering-errors.ts` | 0 | DELETE | U |
| 35 | `modules/cashiering/domain/fx-currency.service.ts` | 2 | DELETE | U — unregistered; paired spec `domain/__tests__/fx-currency.service.spec.ts` must go with it |
| 36 | `modules/cashiering/domain/value-objects/routing-instruction.value-object.ts` | 0 | DELETE | S (orphaned modeling for C32/C33) |
| 37 | `modules/command-center/api/dto/widget-placement.dto.ts` | 0 | DELETE | S |
| 38 | `modules/command-center/api/dto/workspace-response.dto.ts` | 0 | DELETE | S |
| 39 | `modules/command-center/infrastructure/temporal/widget-refresh.workflow.ts` | 0 | **RETAIN** | W — t567 content-read pin (`:41-43,56,321`) + audit-named "exactly one workflow" (audit:574, ARCH-032). *Basis corrected:* **not** reachable via `workflowsPath` default (wrong directory) and never registered (ARCH-032) — W is a test-pin + audit artifact, re-examine at B4.3 (Q8) |
| 40 | `modules/crs-integration/crs-integration.module.ts` | 0 | DELETE | U — vestigial; `crs-front-desk-integration.service.ts` (same dir) is live via `front-office.module.ts:5` (not zero-importer); audit ARCH-029 notes masked cycle |
| 41 | `modules/front-office/application/queries/get-walk-in-defaults/get-walk-in-defaults.handler.ts` | 0 | DELETE | U — route `GET walk-in/defaults` is served by live `WalkInDefaultsService` (`modules/front-office/front-office.controller.ts:27,36-39`); handler/query chain dead |
| 42 | `modules/front-office/application/services/traces-inventory.service.ts` | 0 | **RETAIN** | G — methods map 1:1 to the B4 FE endpoints: `getTraceConfig`↔`GET traces-config`, `autoDeleteExpiredTraces`↔`POST traces/auto-delete`, `syncTracesWithInventory`↔`POST traces/sync-inventory`; unregistered/unrouted, FE client exists (B4). Pair with B4 → Q3. *Was DELETE — G applies; corrected this pass* |
| 43 | `modules/front-office/domain/aggregates/front-office-stay.aggregate.ts` | 0 | **RETAIN** | R — named R8/ARCH-042 deliverable (Phase 6/9 FO domain) |
| 44 | `modules/front-office/domain/value-objects/document-type.value-object.ts` | 0 | DELETE | S |
| 45 | `modules/front-office/domain/value-objects/document-verification-status.value-object.ts` | 0 | DELETE | S |
| 46 | `modules/front-office/domain/value-objects/extracted-identity.value-object.ts` | 0 | DELETE | S |
| 47 | `modules/front-office/domain/value-objects/guest-match-result.value-object.ts` | 0 | DELETE | S |
| 48 | `modules/front-office/domain/value-objects/room-operation.value-object.ts` | 0 | DELETE | S |
| 49 | `modules/front-office/domain/value-objects/walk-in-defaults.value-object.ts` | 0 | DELETE | S (not even the dead handler imports it) |
| 50 | `modules/group-allotment/domain/value-objects/room-type-quantity.value-object.ts` | 0 | DELETE | S — sweep clean (zero refs anywhere) |
| 51 | `modules/group-allotment/index.ts` | 0 | DELETE | B (content live via deep import, e.g. `group-booking.controller.ts:8`; doc/spec hits reference the live module *directory*, not the barrel) |
| 52 | `modules/housekeeping/domain/value-objects/credit-rules-hk-forecast.value-object.ts` | 0 | DELETE | S — pairs with dead B6 feature |
| 53 | `modules/housekeeping/domain/value-objects/hk-membership-scheduling.value-object.ts` | 0 | DELETE | S — pairs with dead B7 feature |
| 54 | `modules/housekeeping/domain/value-objects/turndown-attributes.value-object.ts` | 0 | DELETE | S — pairs with dead B5 feature |
| 55 | `modules/housekeeping/dto.ts` | 0 | DELETE | S — sweep clean |
| 56 | `modules/housekeeping/roles.guard.ts` | 0 | DELETE | U — distinct from live `common/authorization/roles.guard` (registered `APP_GUARD`, `authorization.module.ts:18`) |
| 57 | `modules/inventory/events/inventory-core.events.ts` | 0 | DELETE | S |
| 58 | `modules/inventory/interfaces/inventory-core-service.interface.ts` | 0 | DELETE | S |
| 59 | `modules/inventory/modules/advanced-reservations/dto/advanced-reservations-response.dto.ts` | 0 | DELETE | S (in-module `ADVANCED_RESERVATIONS_AUDIT.md:43` doc listing only) |
| 60 | `modules/inventory/modules/batch-lot/dto/batch-lot-response.dto.ts` | 0 | DELETE | S (`BATCH_LOT_AUDIT.md:45` doc listing only) |
| 61 | `modules/inventory/modules/damage-expiry/dto/damage-expiry-response.dto.ts` | 0 | DELETE | S (`DAMAGE_EXPIRY_AUDIT.md:44` doc listing only) |
| 62 | `modules/inventory/modules/goods-issue/dto/index.ts` | 0 | DELETE | B (siblings live via deep imports; sweep clean) |
| 63 | `modules/inventory/modules/goods-receiving/dto/goods-receiving-response.dto.ts` | 0 | DELETE | S |
| 64 | `modules/inventory/modules/inventory-core/dto/index.ts` | 0 | DELETE | B (sibling `item-crud.dto.ts` C65 also dead — same unit; other siblings live via deep imports; sweep clean) |
| 65 | `modules/inventory/modules/inventory-core/dto/item-crud.dto.ts` | 0 | DELETE | S |
| 66 | `modules/inventory/modules/inventory-intelligence/dto/index.ts` | 0 | DELETE | B (sweep clean) |
| 67 | `modules/inventory/modules/inventory-intelligence/events/inventory-intelligence.events.ts` | 0 | DELETE | S |
| 68 | `modules/inventory/modules/inventory-reports/dto/inventory-reports-response.dto.ts` | 0 | DELETE | S |
| 69 | `modules/inventory/modules/inventory-returns/dto/inventory-returns-response.dto.ts` | 0 | DELETE | S (`INVENTORY_RETURNS_AUDIT.md:51` doc listing only) |
| 70 | `modules/inventory/modules/inventory-settings/dto/inventory-settings-response.dto.ts` | 0 | DELETE | S (`INVENTORY_SETTINGS_AUDIT.md:13` doc listing only) |
| 71 | `modules/inventory/modules/item/dto/item-master-response.dto.ts` | 0 | DELETE | S (inventory controllers live without these DTOs — verified zero refs) |
| 72 | `modules/inventory/modules/physical-inventory/dto/index.ts` | 0 | DELETE | B (sweep clean; `PHYSICAL_INVENTORY_AUDIT.md` references the live response DTO, not the barrel) |
| 73 | `modules/inventory/modules/stock-transfer/dto/index.ts` | 0 | DELETE | B (sweep clean) |
| 74 | `modules/inventory/modules/supplier-management/dto/supplier-management-response.dto.ts` | 0 | DELETE | S (`SUPPLIER_MANAGEMENT_AUDIT.md:48` doc listing only) |
| 75 | `modules/inventory/modules/supply-request/dto/index.ts` | 0 | DELETE | B (sweep clean) |
| 76 | `modules/inventory/modules/warehouse/dto/index.ts` | 0 | DELETE | B (content live via deep import — `warehouse.controller.ts:13`; sweep clean) |
| 77 | `modules/procurement/procurement.module.ts` | 0 | DELETE | U — **Q2 resolved:** the audit itself declares `procurement` UNREGISTERED (`01_CURRENT_ARCHITECTURE_AUDIT.md:131`); no import/registration refs repo-wide (sweep: tree-doc only); subtree (`procurement.controller.ts`, `procurement.service.ts`) transitively dead → delete as unit. ARCH-043 writers unaffected: legacy `purchase_requests` is written by registered `purchasing` (`app.module.ts:39`) and `pr-workflow` (`app.module.ts:95`) |
| 78 | `modules/purchasing/dto.ts` | 0 | DELETE | S (`PurchasingModule` itself is registered — `app.module.ts:39`; sweep clean) |
| 79 | `modules/reservations/api/dto/share-advanced.dto.ts` | 0 | DELETE | S — **Q4 resolved:** live share routes use inline body types (`reservations.controller.ts:365-381`); all 3 DTO symbols referenced nowhere (rg over `apps` = definition only); zero non-import refs (tree-doc only) |
| 80 | `modules/reservations/application/ports/index.ts` | 0 | **RETAIN** | R — Phase-6 port surface (adapters 98-100 RETAIN on same basis); roadmap-cited dir (`reservations-roadmap-v2.md:114`, plan `implementation_plan.md:70`) |
| 81 | `modules/reservations/application/queries/get-email-templates/get-email-templates.handler.ts` | 0 | **RETAIN** | G — FE client chain X1 exists; handler unregistered; route absent. **DDL gap:** SQL targets `email_templates`, which does not exist in `schema.prisma`, any migration, or anywhere else repo-wide (only the handler mentions it) → wiring requires schema work (Q3) |
| 82 | `modules/reservations/application/queries/get-l2b-config/get-l2b-config.handler.ts` | 0 | **RETAIN** | G — FE client B2 exists; handler unregistered; route absent. **DDL gap:** `l2b_config` absent from schema/migrations/repo-wide (only the handler) → wiring requires schema work (Q3) |
| 83 | `modules/reservations/application/queries/get-package-exclusions/get-package-exclusions.handler.ts` | 0 | **RETAIN** | G — FE client X2 exists; handler unregistered; route absent. **DDL gap:** table `packages` exists (`schema.prisma:7574`) but selected columns `is_externally_excluded`, `exclude_from_channels` are absent from the model and all migrations → wiring requires schema work (Q3) |
| 84 | `modules/reservations/application/services/cancellation-policy.service.ts` | 0 | DELETE | U/S — dead **duplicate** of live `domain/services/cancellation-policy.service` (imported by `reservations.module.ts:9`, `cancel-reservation.handler.ts:8`). Doc refs: plan `Antigravity/implementation_plan.md:53` links this path as [NEW] (historical); `opera-detailed-implementation-plan.md:98` specifies the **domain** copy → live one wins |
| 85 | `modules/reservations/constants/reservation.constants.ts` | 0 | DELETE | U — only doc ref is an inventory listing (`phase3-backend-refactoring-report.md:80`); no imports |
| 86 | `modules/reservations/domain/value-objects/auto-cancel-config.value-object.ts` | 0 | DELETE | S — live sweep handler uses inline config (`execute-auto-cancel-sweep.handler.ts:41`); plan-origin ref only (`Antigravity/v2:761`) |
| 87 | `modules/reservations/domain/value-objects/changes-log-override.value-object.ts` | 0 | DELETE | S — pairs with dead B1 feature |
| 88 | `modules/reservations/domain/value-objects/confirmation-email-element.value-object.ts` | 0 | DELETE | S |
| 89 | `modules/reservations/domain/value-objects/enroll-guest-link.value-object.ts` | 0 | DELETE | S (B9 has no backend implementation) |
| 90 | `modules/reservations/domain/value-objects/externally-excluded-package.value-object.ts` | 0 | DELETE | S |
| 91 | `modules/reservations/domain/value-objects/l2b-default-res-type.value-object.ts` | 0 | DELETE | S — orphaned L2B modeling: C82 reads `l2b_config` via `$queryRawUnsafe` with inline shapes and does **not** import this VO; not part of the C82 pair |
| 92 | `modules/reservations/domain/value-objects/l2b-feedback-message.value-object.ts` | 0 | DELETE | S — no FE/BE consumer: FE trio X3 is DELETE (no backend counterpart exists to pair with) |
| 93 | `modules/reservations/domain/value-objects/l2b-room-type-based-charge.value-object.ts` | 0 | DELETE | S — same as C91 (C82 does not import it) |
| 94 | `modules/reservations/domain/value-objects/linked-name-stationery.value-object.ts` | 0 | DELETE | S |
| 95 | `modules/reservations/domain/value-objects/membership-rate-rule.value-object.ts` | 0 | DELETE | S — pairs with dead B8 |
| 96 | `modules/reservations/domain/value-objects/reservation-party.value-object.ts` | 1 | DELETE | S — **Q4 resolved:** ADR-075 invariants are covered at command level by `party-commands.spec.ts:102-154` (≤100 total, rejection codes, update propagation); VO-level `party-invariants.spec.ts` (test importer) is redundant coverage → remove with unit at B4.3; zero non-import refs (tree-doc + plan-origin `Antigravity/v2:101`) |
| 97 | `modules/reservations/domain/value-objects/share-defaults.value-object.ts` | 0 | DELETE | S — **Q4 resolved:** live `get-share-defaults.handler` returns raw shapes without the VO; zero refs anywhere |
| 98 | `modules/reservations/infrastructure/adapters/prisma-front-office-handoff.adapter.ts` | 0 | **RETAIN** | R — explicitly named in R8/ARCH-042; Phase-6 deliverable (availability docs record it orphaned-pending-Phase-6, e.g. `phase-6/01_FORENSIC_AUDIT.md` G-11 — consistent with R) |
| 99 | `modules/reservations/infrastructure/adapters/prisma-guest-profile.adapter.ts` | 0 | **RETAIN** | R |
| 100 | `modules/reservations/infrastructure/adapters/prisma-inventory-reservation.adapter.ts` | 1 | **RETAIN** | R — test pins: `reservation-hold-mechanism.postgres.spec.ts:2` (import) **and** cross-app `apps/web/.../t542-dead-artifacts.test.ts:97` (`fs.existsSync` must stay true); heavily cited in Phase-6 docs (L-7 register) |
| 101 | `modules/reservations/permissions/index.ts` | 0 | DELETE | B (content live via deep import: `front-office.controller.ts:11`, specs; sweep clean) |
| 102 | `modules/shared/queue.consumers.ts` | 0 | **RETAIN** | W — **pin-only**: t567 content-read (`:40,53,356,365`). Runtime-dead per audit:583 ("provided by no module", queue not registered) + `02_OPEN_EVIDENCE_RESOLUTION.md:960` ("0 module imports"); stale "Imported by" comments in `packages/analytics/src/pipelines/triggers/on-night-audit-complete.ts:6` and `on-folio-post.ts:6` (doc debt, §5.4). Re-examine with pin at B4.3 (Q8) |
| 103 | `platform/configuration/dto/setting-response.dto.ts` | 0 | DELETE | S |
| 104 | `platform/registry/dto/register-activity.dto.ts` | 0 | DELETE | S |
| 105 | `platform/registry/dto/register-command.dto.ts` | 0 | DELETE | S |
| 106 | `platform/registry/dto/register-event.dto.ts` | 0 | DELETE | S |
| 107 | `platform/registry/dto/register-module.dto.ts` | 0 | DELETE | S |
| 108 | `platform/registry/dto/register-query.dto.ts` | 0 | DELETE | S |
| 109 | `platform/registry/dto/register-widget.dto.ts` | 0 | DELETE | S |
| 110 | `platform/registry/dto/register-workflow.dto.ts` | 0 | DELETE | S |

### 4.1 Coupled pairs, wiring gaps, and special rows

- **Five G/P pairs — one decision each (Q3).** For each pair the backend implementation exists but is unregistered/unrouted while a closed FE client chain exists; both sides are RETAIN until the owner wires or deletes the pair:

| Pair | Backend (Class C) | FE unit | Endpoint (404 today) | DDL gap? |
|---|---|---|---|---|
| Email templates | C81 | §3.1 X1 | `GET /reservations/email-templates` | **Yes** — `email_templates` table absent from schema/migrations/repo |
| L2B config | C82 | B2 | `GET /reservations/l2b-config` | **Yes** — `l2b_config` table absent |
| Package exclusions | C83 | §3.1 X2 | `GET /reservations/package-exclusions` | **Yes** — columns absent on existing `packages` table |
| Traces extended ops | C42 | B4 | `GET traces-config` + 2 POSTs | No (service uses Prisma already) |
| Batch CC auth | C31 | §3.1 X4 | `POST /cashiering/batch-cc-authorization` | No |

  Wiring the first three requires schema/DDL work, which Phase 3 forbids (no schema changes) — the owner must weigh "wire later (needs migration)" vs "delete both sides at B4.3".

- **Deletion-unit closure (B4.3):** every FE unit also removes its dead export lines from `features/*/hooks/index.ts`, `features/*/api/index.ts`, and feature barrels (`features/reservations/index.ts` for A6, batch-cc, email/package lists). Backend G pairs are deleted as full pairs only (never one side).
- **C41:** the live walk-in-defaults route proves the handler is a dead parallel path, not a wiring gap → DELETE stands.
- **C81–C83 handler registration proof:** filename stems absent from `reservations.module.ts` (44 commands + 18 queries registered — assessment §2.11); no other registration site repo-wide; sweep shows no string references beyond the tree listing.

## 5. Drift, pins, and risks

### 5.1 82 → 110 drift
The audit recorded 82 at audit time; current scan finds 110. Two independent blockers prevent an item-level diff: (a) the audit's per-file list was never persisted (only category descriptions in §10.1.4/§13); (b) **git-history dating is impossible** — the repository has 5 commits and the working tree is largely untracked (`share-advanced.dto.ts`, the audit document itself, etc. are `??`), so file-add dates cannot be compared against the audit date (§1.7). Delta is *consistent with* post-audit additions (S2 shares/party assets, config panels, inventory scaffold DTOs) but cannot be proven here. Current evidence is authoritative for B4.3; every deletion is re-verified anyway.

### 5.2 Transitive-dependents warning (B4.3)
The scan counts **direct** importers. Files whose only importer is a zero-listed file are NOT in the list (e.g. `get-*.query.ts` files under C81–C83, `crs-front-desk-integration`'s neighbours, procurement's controller/service, `inventory-core/dto/item-crud.dto.ts`'s siblings-in-unit). **Deletion units must be closed transitively** and re-verifed at deletion time.

### 5.3 String-path / test / doc pins found
Spec pins (`spec-string-pin-sweep.js`, 10 hits): `t567-async-scoping.spec.ts` content-reads `common/queue/worker-base.ts` (`:49,:58,:356`), `modules/shared/queue.consumers.ts` (`:40,:53,:365`), `widget-refresh.workflow.ts` (`:41-43,:56,:321`) and its activity; `t564-api-isolation.spec.ts:16` imports `partition-router.middleware`; `ws-n-scan-gates.spec.ts:332,334` pins `main-worker.ts`; `t521-availability-write-dto.spec.ts:28` comments `main.ts`; `fx-currency.service.spec.ts` imports C35 (×2); `party-invariants.spec.ts` imports C96; `reservation-hold-mechanism.postgres.spec.ts:2` imports C100; `cancellation-penalty.spec.ts` line 1 resolves to the **domain** copy (false positive for C84). Cross-app pin: `apps/web/features/availability/__tests__/t542-dead-artifacts.test.ts:97` requires C100 to exist. Temporal workflows are loaded via `TEMPORAL_WORKFLOWS_PATH` string — **not import-reachable** (this is why C28 is RETAIN W).

#### 5.3.1 Whole-repo sweep result
219 kept hit-lines across 97/110 files (report in temp, §1.6). Hits are overwhelmingly (a) `PROJECT_STRUCTURE.md` tree listings `[tree-doc]` — informational, (b) audit/architecture docs that **agree** with the verdict (audit:254 C20, audit:318 C25/C24, audit:583/584 C102/C11, OER:960/1141, forensic:713 C24), (c) plan-origin references in `docs/Antigravity/*` (C79/C84/C86/C96 + ports dir), (d) doc listings of live inventory dirs (not the barrels). **13 files have zero references anywhere outside this ledger:** `common/base/index.ts`, `infrastructure/temporal/activities/index.ts`, `modules/group-allotment/domain/value-objects/room-type-quantity.value-object.ts`, `modules/housekeeping/dto.ts`, `modules/purchasing/dto.ts`, `modules/reservations/permissions/index.ts`, and the 7 inventory `dto/index.ts` barrels (C62/C64/C66/C72/C73/C75/C76).

### 5.4 Documentation conflicts / doc debt for owner
1. **AGENTS.md** (`:48`) states RLS runs via `property-scope.guard.ts` **and** `partition-router.middleware.ts` — the middleware is unregistered (C24), and `availability-phase3-reservation-lifecycle-forensic-audit.md:713` already declares it dead. Ruling Q1: wire it, or delete it and correct AGENTS.md (phase-5 availability docs also list it as "located in repo" — update those too).
2. **ARCH-043 procurement claim — RESOLVED (Q2):** the same audit declares `crs-integration`, `procurement` UNREGISTERED (`:131`); live legacy writers are `purchasing` + `pr-workflow` (both registered) + housekeeping services. C77 deletion does not affect ARCH-043's write paths (§4 row 77).
3. **Stale comments:** `packages/analytics/src/pipelines/triggers/on-night-audit-complete.ts:6` and `on-folio-post.ts:6` claim "Imported by apps/api/modules/shared/queue.consumers.ts" — no such import exists (C102 runtime-dead). Doc debt for Phase 11, not a deletion blocker.
4. **`PROJECT_STRUCTURE.md`** is a full file tree listing — every deletion will make it stale; regenerate at Phase 11 (informational, not a retention reason).

## 6. Acceptance-criterion evidence map

| Criterion (source) | Evidence |
|---|---|
| Ledger drafted (B3.5 row) | This document — classes A, B, C with explicit RETAIN/DELETE per item + §3.1 pairs |
| Importer-count evidence (B3.5 row) | §1 method; §2 counts/render sites; §4 all 110 rows with importer counts; named-member 8/8 verification; FE unit closure checks (§1.2) |
| Owner review queued (B3.5 row) | §7 — Q1–Q8 with dispositions (2 resolved by evidence, 3 resolved-technical, 3 open gates) |
| Deletion NOT performed (Phase-3 exit gate) | Zero file removals; git baseline preserved: 751 pre-existing `git status` entries → **751** after (new file sits inside the already-listed untracked `?? docs/reservations/` entry); 0 stashes |
| Evidence recorded, not a test (B3.5 focused-tests column) | No tests added or modified; verification = read-only scans (§8) |
| Deletion deferred with authorization + re-verification (R8/B4.3) | §7 Q8 + §5.2 re-verification requirement restated for B4.3 |
| Remediation pass consistency (this pass) | §0 log; §1.6 sweep (13 zero-ref files); coupling/rule corrections (B2/B4/C26/C27/C31/C42); endpoint proofs re-run (§8) |

## 7. Owner review queue

| # | Decision needed | Rows affected | Status after remediation pass |
|---|---|---|---|
| Q1 | Wire or delete `partition-router.middleware` + correct AGENTS.md (and phase-5 doc references) | C24 | **OPEN** (M) — evidence complete: registration search negative at all sites; docs disagree with each other (AGENTS "active" vs forensic "dead"); recommendation: delete + fix docs unless middleware wiring is an explicit intent |
| Q2 | Confirm procurement write path before authorizing subtree deletion | C77 | **RESOLVED by evidence** — audit:131 declares it unregistered; live `purchase_requests` writers = `purchasing` (app.module:39) + `pr-workflow` (app.module:95); ARCH-043 unaffected. DELETE technical confirmed → authorization at Q8 |
| Q3 | Wiring-gap pairs: wire (first three need schema/DDL work — see §4.1) or delete **both sides** | C81+X1, C82+B2, C83+X2, C42+B4, C31+X4 | **OPEN** (G/P) — evidence complete incl. DDL gaps; 5 pairs, one decision each; recommendation recorded per pair at review |
| Q4 | Recent S2 shares/parties assets: confirm deletion (incl. paired specs) | C79, C96, C97 | **RESOLVED by evidence** — live routes use inline body types; ADR-075 invariants covered by `party-commands.spec.ts`; handlers don't use the VOs; origin = unadopted plan. DELETE technical confirmed → authorization (incl. `party-invariants.spec.ts`) at Q8 |
| Q5 | Cashiering feature intent (daily-routing) | C32–C36 (C31 moved to Q3) | **RESOLVED-technical** — no FE side exists for daily-routing; plan origin `Antigravity/v2` P2-4/P2-5 has unchecked boxes and is referenced by no authoritative source. DELETE technical → authorization at Q8 |
| Q6 | Inventory scaffold DTO/barrel deletion approval (legacy-domain caution) | C59–C76 | **RESOLVED-technical** — barrels dead (B, deep-import proofs), DTOs unreferenced (in-module `*_AUDIT.md` hits are doc inventories only); controllers live without them. DELETE technical → authorization at Q8 |
| Q7 | Extras disposition | §3 X3/X5/X6 + email/package lists | **RESOLVED-technical** — see §3.1: X1/X2 RETAIN (join C81/C83 pairs), X3/X5 DELETE, X6 DELETE-candidate (partial store unit). Authorization at Q8 |
| Q8 | Approve full DELETE set for B4.3 execution (with per-item zero-reference re-check + suite green) | all DELETE rows; also re-examine W-pin-only rows C11/C39/C102 | **OPEN** (inherent gate — B4.3's authorization) |

## 8. Verification log (commands, observed results)

```text
node <temp>/zero-importer-scan.js            (re-run this pass)
  → 1320 non-test files; 110 zero-non-test-importer; named 8/8 present; identical to first run

node <temp>/spec-string-pin-sweep.js         (re-run this pass)
  → same 10 spec pins as first run (§5.3)

node <temp>/full-repo-sweep.js               (new this pass; 433 patterns, all file types)
  → 219 kept hit-lines; 97/110 files referenced outside ledger; 13 files zero-ref (§5.3.1)

ledger path table ↔ scan list (PowerShell compare, slash-normalized)
  → 110 = 110, 0 diffs

rg route strings (all 9 B-endpoints + X1-X6 + extras) over apps/api/src (controllers + all files)
  → absent everywhere; present live: auto-cancel/sweep :521, traces CRUD :152-162,
    tax-config :146, walk-in/defaults :36, housekeeping sla/status-flow, pbx-config

prefix chain: services/api.ts:1,81 + next.config.js:25,57-63 + main.ts:21 → 404 proof (§1.3)

registration sweeps:
  partition-router: main.ts / app.module / configure() → only CorrelationIdMiddleware (common.module.ts:57-58)
  G-pair handlers: filename stems absent from reservations.module.ts; cashiering.controller route list enumerated
  procurement: repo-wide "ProcurementModule|procurement.service" → subtree only; audit:131 UNREGISTERED
  purchase_requests writers: procurement + purchasing + pr-workflow (last two registered app.module:39,95)

DDL cross-check: schema.prisma + 3 migration trees + repo-wide →
  l2b_config: only handler | email_templates: only handler | packages: model at :7574 but
  is_externally_excluded/exclude_from_channels columns absent

FE unit closure: rg importers of 14 api files + 11 hooks over apps/web
  → each unit = panel + hook + api (+ feature index.ts barrels)

git: rev-list --count HEAD → 5; ls-files of representative zero files + audit doc → untracked (??);
  git status --porcelain | Measure-Object → 751 entries (unchanged); git stash list → 0

rg "from .+activities|reservations/events" apps/web          → 1 edge (ActivitiesTab→FunctionDiary); barrel 0
rg "EditReservationModal|ReservationDetailModal|ReservationToolbar|ReservationWorkspaceNav" apps/web
  → definition-only; render-site rg "<…" → 0 for all except ReservationSidebarNav (workspace-layout:34)
rg "GetEmailTemplatesQuery|GetPackageExclusionsQuery|GetL2BConfigQuery" apps/api/src → own dirs only (unwired)
rg "GetWalkInDefaults|walk-in/defaults" → live service path in front-office.controller (:27,:36-39)
```

Scan/sweep scripts intentionally left in temp (not committed) — method is fully specified in §1 for reproduction.

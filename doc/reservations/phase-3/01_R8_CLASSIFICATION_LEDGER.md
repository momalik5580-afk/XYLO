# B3.5 — R8 Classification Ledger (RETAIN/DELETE, classify-only)

**Status:** LEDGER DRAFTED — OWNER REVIEW QUEUED. **No deletion performed** (execution is B4.3 and requires authorization + zero-reference re-verification at deletion time).

**Authority (single source):** `docs/reservations/phase-1/02_PHASE1_CLOSURE_AND_ROADMAP.md`

- **R8 row** (findings table): dead/legacy candidates — `components/reservations/events/` (23 files), `activities/`, 2 modals, `ReservationToolbar`, 9 orphan config endpoints that would 404, duplicate nav; ARCH-042: 82 zero-importer files repo-wide (incl. Reservations Phase-6 port adapters). Status `SUSPECTED / classify-only`. Resolution: *Classify → RETAIN/DELETE ledger; deletion only in Phase 4 after zero-reference evidence + authorization. Tests: importer-count evidence per item at deletion time.* Batch split: classify **Phase 3 / B3.5**; delete **Phase 4 / B4.3**.
- **B3.5 row:** scope = *R8 classification (no deletion): RETAIN/DELETE ledger for dead components, orphan endpoints, ARCH-042 zero-importer list*; acceptance = *ledger drafted with importer-count evidence; owner review queued*; focused tests = *evidence recorded, not a test*; deps = B3.1.
- **Phase 3 exit gate:** *B3.5 ledger exists (deletion NOT performed).*
- **B4.3 row:** *Execute RETAIN/DELETE ledger (R8) with authorization; deletion only after zero-reference re-verification at deletion time.*

**Supporting sources:** `docs/reservations/phase-1/01_PHASE1_ASSESSMENT_REPORT.md` §2 item 18 (cited as "§2.18") and risk row R8; `docs/architecture/01_CURRENT_ARCHITECTURE_AUDIT.md` §10.1 item 4 + §13 ARCH-042 (audit method §1.2).

**Conflict/discrepancy check vs other approved sources:** `docs/enterprise/reservations-roadmap-v2.md` (tracking layer) contains **no B3.5 definition and no conflict** — its Phase 11 exit gate (*RETAIN/TRANSFORM/MERGE/ARCHIVE/DISCARD ledger **executed***) is execution-scope (B4.3/Phase 11), not classification. Minor source defects recorded, not blocking: (a) assessment has no `§2.18` heading — the cited content is §2 item 18 (verified); (b) R8's reported counts (`23` event files, `9` config endpoints, `82` zero-importer files) were `[R]`-marked reported values — current counts differ (see §3/§4/§5 drift notes).

---

## 1. Method (evidence, not tests)

No product or test code touched. Evidence collected 2026-10-10 by read-only scans:

1. **Zero-importer scan (backend, ARCH-042 class):** re-implementation of audit §1.2 method over `apps/api/src` — enumerate non-test `.ts` files (exclude `*.spec.ts`, `*.test.ts`, `__tests__/`, `*.d.ts`); extract every `from '…'`, `import '…'`, `import('…')`, `require('…')` specifier; resolve relative and `@/*` (→ `src/*`) specifiers to absolute paths (`+ .ts`, `/index.ts`); count **non-test** importers per file; list files with count 0. Script kept out of the repo (temp): `C:\Users\Pro\AppData\Local\Temp\opencode\zero-importer-scan.js`.
2. **Importer/render counts (frontend, R8 class):** `rg` for import specifiers and JSX render sites per candidate path/symbol across `apps/web`.
3. **Endpoint existence (orphan-endpoint class):** each frontend endpoint literal matched against backend `@Controller`/`@Get`/`@Post`/`@Put`/`@Delete` route strings (`rg` over `apps/api/src`).
4. **Registration checks:** module/provider references in `app.module.ts`, `*.module.ts` (CQRS handlers are registered by manual import in module files — AGENTS.md).
5. **False-positive sweep:** (a) string-path/`require.resolve`/config-path wiring check (`workflowsPath`); (b) spec-file string-reference sweep over all 110 zero-importer paths (`spec-string-pin-sweep.js`, temp) — found only the pins named in §5.3.

**Scan result:** 1320 non-test `.ts` files in `apps/api/src`; **110 files with zero non-test importers** (audit recorded 82 at audit time — drift note §5.1); of the 110, 4 have test-only importers (`partition-router.middleware` 1, `fx-currency.service` 2, `reservation-party.value-object` 1, `prisma-inventory-reservation.adapter` 1); remaining 106 are referenced by nothing at all (after sweep, except string pins §5.3).

**Named ARCH-042 members verified present in the current scan (8/8):** `prisma-front-office-handoff.adapter.ts`, `prisma-guest-profile.adapter.ts`, `prisma-inventory-reservation.adapter.ts`, `front-office-stay.aggregate.ts`, `tenant-injection.extension.ts`, and the 3 unwired reservations handlers (`get-l2b-config`, `get-email-templates`, `get-package-exclusions`).

**Classification rules used (applied uniformly; owner review may override):**

| Code | Rule | Verdict |
|---|---|---|
| E | NestJS entrypoint — zero importers by design (executed by runtime) | RETAIN |
| W | Wired by non-import mechanism (Temporal `workflowsPath` string; t567 path pins) | RETAIN |
| R | Roadmap deliverable (Reservations Phase-6 ports/aggregate) — R8 explicitly names the adapters | RETAIN |
| G | Wiring gap — backend handler exists but unregistered **and** a frontend API client exists | RETAIN (owner: wire or delete pair) |
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

Evidence per row: frontend panel mount state (0 render sites unless noted) + backend route existence (`rg` over all `*.controller.ts` route strings).

| # | Config panel (mount state) | Client files (hook → api) | Endpoint | Backend route evidence | Verdict |
|---|---|---|---|---|---|
| B1 | `ChangesLogConfigPanel` (0 renders) | `features/reservations/hooks/use-changes-log-config.ts` → `api/changes-log-config.api.ts` | `GET /reservations/changes-log-config` | **absent** (no `changes-log-config` route in `apps/api/src`) | **DELETE** (3 files) |
| B2 | `L2BConfigDisplay` (0 renders) | `use-l2b-config.ts` → `l2b-config.api.ts` | `GET /reservations/l2b-config` | **absent** — `GetL2BConfigHandler` exists but is **unregistered** (C3 row) | **DELETE** (3 files) — coupled to C3, owner decides pair |
| B3 | `AutoCancelSweepButton` (0 renders) | `use-auto-cancel.ts` → `auto-cancel.api.ts` | `GET /reservations/auto-cancel/config` | **absent** — only `POST auto-cancel/sweep` exists (`reservations.controller.ts:521`, live) | **DELETE** (3 files; sweep backend stays live/headless) |
| B4 | `TracesInventoryPanel` (0 renders) | `use-traces-inventory.ts` → `traces-inventory.api.ts` | `GET /front-office/traces-config`, `POST /front-office/traces/auto-delete`, `POST /front-office/traces/sync-inventory` | **all absent** (FO controller has `traces` CRUD only, `:152-162`) | **DELETE** (3 files) |
| B5 | `TurndownAttributesConfig` (0 renders) | `use-turndown-attributes.ts` → `turndown-attributes.api.ts` | `GET/PUT /housekeeping/turndown-attributes` | **absent** | **DELETE** (3 files) |
| B6 | `CreditRulesConfig` (0 renders) | `use-credit-rules.ts` → `credit-rules.api.ts` | `GET/PUT /housekeeping/credit-rules` | **absent** | **DELETE** (3 files) |
| B7 | `MembershipSchedulingConfig` (0 renders) | `use-membership-scheduling.ts` → `membership-scheduling.api.ts` | `GET/PUT /housekeeping/membership-scheduling` | **absent** | **DELETE** (3 files) |
| B8 | `MembershipRateRuleConfig` (0 renders) | `use-membership-rate-rule.ts` → `membership-rate-rule.api.ts` | `GET/PUT /settings/membership-rate-rule` | **absent** | **DELETE** (3 files) |
| B9 | `LoyaltyEnrollmentConfig` (0 renders) | `use-loyalty-enrollment.ts` → `loyalty-enrollment.api.ts` | `GET/PUT /settings/loyalty-enrollment` | **absent** | **DELETE** (3 files) |

**Count reconciliation:** exactly **9** unmounted config panels whose primary config `GET` has no backend route — reproduces the reported "9 orphan config endpoints that would 404" (`[R]`) under the one-config-GET-per-panel reading. Auxiliary endpoints of the same panels (PUT verbs, `traces/auto-delete`, `traces/sync-inventory`) are in the same deletion units.

**Verified live (excluded — do NOT delete):** `GET /front-office/tax-config` (`front-office.controller.ts:146`, consumed by live `CheckoutFlowModal`); `GET/PUT /housekeeping/config/sla` (`housekeeping.controller.ts:236,241`); `GET/PUT /housekeeping/config/status-flow` (`:246,251`); `GET/PUT /pbx-config/:id` (`pbx-config.controller.ts:4`).

**Observed extras (outside the reported 9; recorded, not classified as B-rows):**
- `/housekeeping/linen/room-configs` (GET/POST/PUT/DELETE) — no backend route; called only from `housekeepingStore.ts:1355-1388`; store actions have **0 UI callers** → candidate for partial store-method removal at B4.3 (owner).
- `/settings/property-description` (GET/PUT) — no backend route; `PropertyDescriptionForm.tsx` has 0 render sites — same orphan pattern, not config-named (owner: fold into B4.3 or Phase 11).
- `EmailTemplatesList` / `PackageExclusionsList` / `L2BFeedbackMessagesList` — 0 render sites each; their backend handlers are C3/C11 rows (coupled decision).

## 4. Class C — ARCH-042 zero-importer list (backend, current scan)

**Totals:** 1320 non-test files scanned; **110 zero-non-test-importer** (audit-time: 82 → +28 drift, §5.1). Verdicts: **17 RETAIN, 93 DELETE.** Non-test importers = 0 for every row by construction; test importers shown.

| # | Path (`apps/api/src/…`) | Test | V | Basis |
|---|---|---|---|---|
| 1 | `common/audit/audit.interceptor.ts` | 0 | DELETE | U |
| 2 | `common/authorization/index.ts` | 0 | DELETE | B (content live via deep import) |
| 3 | `common/base/index.ts` | 0 | DELETE | B |
| 4 | `common/cqrs/index.ts` | 0 | DELETE | B (e.g. `cashiering.service.ts:2` deep-imports `common/cqrs/command-bus`) |
| 5 | `common/database/repositories/index.ts` | 0 | DELETE | B |
| 6 | `common/events/base-event.ts` | 0 | DELETE | U |
| 7 | `common/events/event-handler.interface.ts` | 0 | DELETE | U |
| 8 | `common/helpers/async-helpers.ts` | 0 | DELETE | U |
| 9 | `common/helpers/crypto.helper.ts` | 0 | DELETE | U |
| 10 | `common/outbox/index.ts` | 0 | DELETE | B (content live: `common.module.ts:17`, `event.module.ts:6-7`) |
| 11 | `common/queue/worker-base.ts` | 0 | **RETAIN** | W — t567 path pin (`t567-async-scoping.spec.ts:49`) |
| 12 | `common/response/pagination.dto.ts` | 0 | DELETE | S |
| 13 | `common/response/response.types.ts` | 0 | DELETE | S |
| 14 | `common/storage/storage.types.ts` | 0 | DELETE | S |
| 15 | `common/swagger/swagger.constants.ts` | 0 | DELETE | U |
| 16 | `common/testing/fixtures.ts` | 0 | DELETE | U |
| 17 | `common/testing/testing.module.ts` | 0 | DELETE | U |
| 18 | `common/validation/cross-field.validator.ts` | 0 | DELETE | U |
| 19 | `common/validation/unique.validator.ts` | 0 | DELETE | U |
| 20 | `core/filters/http-exception.filter.ts` | 0 | DELETE | U — no registration anywhere incl. `main.ts` |
| 21 | `core/guards/compliance.guard.ts` | 0 | DELETE | U |
| 22 | `core/guards/partition-access.guard.ts` | 0 | DELETE | U |
| 23 | `core/interceptors/audit.interceptor.ts` | 0 | DELETE | U |
| 24 | `core/middleware/partition-router.middleware.ts` | 1 | **RETAIN** | M — unregistered (only `t564` imports it) **but** AGENTS.md cites it as active RLS architecture → owner ruling: wire or delete + fix docs |
| 25 | `infrastructure/prisma/extensions/tenant-injection.extension.ts` | 0 | DELETE | L (ARCH-042; superseded by ALS/`property-scope.guard` path) |
| 26 | `infrastructure/temporal/activities/index.ts` | 0 | **RETAIN** | W — loaded via `TEMPORAL_WORKFLOWS_PATH` string (`temporal-config.service.ts:41`) |
| 27 | `infrastructure/temporal/index.ts` | 0 | **RETAIN** | W (same) |
| 28 | `infrastructure/temporal/workflows/index.ts` | 0 | **RETAIN** | W (same; default `./dist/infrastructure/temporal/workflows`) |
| 29 | `main-worker.ts` | 0 | **RETAIN** | E (entrypoint; also pinned by `ws-n-scan-gates.spec.ts:332`) |
| 30 | `main.ts` | 0 | **RETAIN** | E (entrypoint) |
| 31 | `modules/cashiering/application/commands/batch-cc-authorization/batch-cc-authorization.handler.ts` | 0 | DELETE | U — no controller route anywhere; FE client `batch-cc.api.ts` exists but `BatchCCAuthButton` has 0 renders (feature dead both sides; owner note) |
| 32 | `modules/cashiering/application/commands/create-daily-routing/create-daily-routing.handler.ts` | 0 | DELETE | U — no route, no FE reference |
| 33 | `modules/cashiering/application/queries/get-routing-for-day/get-routing-for-day.handler.ts` | 0 | DELETE | U (same) |
| 34 | `modules/cashiering/domain/errors/cashiering-errors.ts` | 0 | DELETE | U |
| 35 | `modules/cashiering/domain/fx-currency.service.ts` | 2 | DELETE | U — unregistered; paired spec `domain/__tests__/fx-currency.service.spec.ts` must go with it |
| 36 | `modules/cashiering/domain/value-objects/routing-instruction.value-object.ts` | 0 | DELETE | S |
| 37 | `modules/command-center/api/dto/widget-placement.dto.ts` | 0 | DELETE | S |
| 38 | `modules/command-center/api/dto/workspace-response.dto.ts` | 0 | DELETE | S |
| 39 | `modules/command-center/infrastructure/temporal/widget-refresh.workflow.ts` | 0 | **RETAIN** | W — `workflowsPath` string + t567 pin (`:43,56`) |
| 40 | `modules/crs-integration/crs-integration.module.ts` | 0 | DELETE | U — vestigial; `crs-front-desk-integration.service.ts` (same dir) is live via `front-office.module.ts:5` (not zero-importer) |
| 41 | `modules/front-office/application/queries/get-walk-in-defaults/get-walk-in-defaults.handler.ts` | 0 | DELETE | U — route `GET walk-in/defaults` is served by live `WalkInDefaultsService` (`front-office.controller.ts:27,36-39`); handler/query chain dead |
| 42 | `modules/front-office/application/services/traces-inventory.service.ts` | 0 | DELETE | U — no route; FE panel unmounted (B4) |
| 43 | `modules/front-office/domain/aggregates/front-office-stay.aggregate.ts` | 0 | **RETAIN** | R — named R8/ARCH-042 deliverable (Phase 6/9 FO domain) |
| 44 | `modules/front-office/domain/value-objects/document-type.value-object.ts` | 0 | DELETE | S |
| 45 | `modules/front-office/domain/value-objects/document-verification-status.value-object.ts` | 0 | DELETE | S |
| 46 | `modules/front-office/domain/value-objects/extracted-identity.value-object.ts` | 0 | DELETE | S |
| 47 | `modules/front-office/domain/value-objects/guest-match-result.value-object.ts` | 0 | DELETE | S |
| 48 | `modules/front-office/domain/value-objects/room-operation.value-object.ts` | 0 | DELETE | S |
| 49 | `modules/front-office/domain/value-objects/walk-in-defaults.value-object.ts` | 0 | DELETE | S (not even the dead handler imports it) |
| 50 | `modules/group-allotment/domain/value-objects/room-type-quantity.value-object.ts` | 0 | DELETE | S |
| 51 | `modules/group-allotment/index.ts` | 0 | DELETE | B (content live via deep import, e.g. `group-booking.controller.ts:8`) |
| 52 | `modules/housekeeping/domain/value-objects/credit-rules-hk-forecast.value-object.ts` | 0 | DELETE | S — pairs with dead B6 feature |
| 53 | `modules/housekeeping/domain/value-objects/hk-membership-scheduling.value-object.ts` | 0 | DELETE | S — pairs with dead B7 feature |
| 54 | `modules/housekeeping/domain/value-objects/turndown-attributes.value-object.ts` | 0 | DELETE | S — pairs with dead B5 feature |
| 55 | `modules/housekeeping/dto.ts` | 0 | DELETE | S |
| 56 | `modules/housekeeping/roles.guard.ts` | 0 | DELETE | U — distinct from live `common/authorization/roles.guard` (registered `APP_GUARD`, `authorization.module.ts:18`) |
| 57 | `modules/inventory/events/inventory-core.events.ts` | 0 | DELETE | S |
| 58 | `modules/inventory/interfaces/inventory-core-service.interface.ts` | 0 | DELETE | S |
| 59 | `modules/inventory/modules/advanced-reservations/dto/advanced-reservations-response.dto.ts` | 0 | DELETE | S |
| 60 | `modules/inventory/modules/batch-lot/dto/batch-lot-response.dto.ts` | 0 | DELETE | S |
| 61 | `modules/inventory/modules/damage-expiry/dto/damage-expiry-response.dto.ts` | 0 | DELETE | S |
| 62 | `modules/inventory/modules/goods-issue/dto/index.ts` | 0 | DELETE | B |
| 63 | `modules/inventory/modules/goods-receiving/dto/goods-receiving-response.dto.ts` | 0 | DELETE | S |
| 64 | `modules/inventory/modules/inventory-core/dto/index.ts` | 0 | DELETE | B |
| 65 | `modules/inventory/modules/inventory-core/dto/item-crud.dto.ts` | 0 | DELETE | S |
| 66 | `modules/inventory/modules/inventory-intelligence/dto/index.ts` | 0 | DELETE | B |
| 67 | `modules/inventory/modules/inventory-intelligence/events/inventory-intelligence.events.ts` | 0 | DELETE | S |
| 68 | `modules/inventory/modules/inventory-reports/dto/inventory-reports-response.dto.ts` | 0 | DELETE | S |
| 69 | `modules/inventory/modules/inventory-returns/dto/inventory-returns-response.dto.ts` | 0 | DELETE | S |
| 70 | `modules/inventory/modules/inventory-settings/dto/inventory-settings-response.dto.ts` | 0 | DELETE | S |
| 71 | `modules/inventory/modules/item/dto/item-master-response.dto.ts` | 0 | DELETE | S (inventory controllers live without these DTOs — verified zero refs) |
| 72 | `modules/inventory/modules/physical-inventory/dto/index.ts` | 0 | DELETE | B |
| 73 | `modules/inventory/modules/stock-transfer/dto/index.ts` | 0 | DELETE | B |
| 74 | `modules/inventory/modules/supplier-management/dto/supplier-management-response.dto.ts` | 0 | DELETE | S |
| 75 | `modules/inventory/modules/supply-request/dto/index.ts` | 0 | DELETE | B |
| 76 | `modules/inventory/modules/warehouse/dto/index.ts` | 0 | DELETE | B |
| 77 | `modules/procurement/procurement.module.ts` | 0 | DELETE | U — no reference in any style (incl. `app.module.ts`); subtree (`procurement.controller.ts`, `procurement.service.ts`) transitively dead → delete as unit. **Flag:** tensions with audit ARCH-043 "procurement writes legacy inventory" — owner confirm at review |
| 78 | `modules/purchasing/dto.ts` | 0 | DELETE | S (`PurchasingModule` itself is registered — `app.module.ts:39`) |
| 79 | `modules/reservations/api/dto/share-advanced.dto.ts` | 0 | DELETE | S — S2 shares asset, unused by shipped commands; owner confirm |
| 80 | `modules/reservations/application/ports/index.ts` | 0 | **RETAIN** | R — Phase-6 port surface (adapters 98-100 RETAIN on same basis) |
| 81 | `modules/reservations/application/queries/get-email-templates/get-email-templates.handler.ts` | 0 | **RETAIN** | G — FE client `email-templates.api.ts` + unmounted `EmailTemplatesList`; owner: wire route or delete pair |
| 82 | `modules/reservations/application/queries/get-l2b-config/get-l2b-config.handler.ts` | 0 | **RETAIN** | G — FE client `l2b-config.api.ts` (B2); coupled with B2 verdict |
| 83 | `modules/reservations/application/queries/get-package-exclusions/get-package-exclusions.handler.ts` | 0 | **RETAIN** | G — FE client `package-exclusions.api.ts` + unmounted `PackageExclusionsList` |
| 84 | `modules/reservations/application/services/cancellation-policy.service.ts` | 0 | DELETE | U/S — dead **duplicate** of live `domain/services/cancellation-policy.service` (imported by `reservations.module.ts:9`, `cancel-reservation.handler.ts:8`) |
| 85 | `modules/reservations/constants/reservation.constants.ts` | 0 | DELETE | U |
| 86 | `modules/reservations/domain/value-objects/auto-cancel-config.value-object.ts` | 0 | DELETE | S — live sweep handler uses inline config (`execute-auto-cancel-sweep.handler.ts:41`) |
| 87 | `modules/reservations/domain/value-objects/changes-log-override.value-object.ts` | 0 | DELETE | S — pairs with dead B1 feature |
| 88 | `modules/reservations/domain/value-objects/confirmation-email-element.value-object.ts` | 0 | DELETE | S |
| 89 | `modules/reservations/domain/value-objects/enroll-guest-link.value-object.ts` | 0 | DELETE | S |
| 90 | `modules/reservations/domain/value-objects/externally-excluded-package.value-object.ts` | 0 | DELETE | S |
| 91 | `modules/reservations/domain/value-objects/l2b-default-res-type.value-object.ts` | 0 | DELETE | S — pairs with B2 |
| 92 | `modules/reservations/domain/value-objects/l2b-feedback-message.value-object.ts` | 0 | DELETE | S — FE `L2BFeedbackMessagesList` also 0 renders |
| 93 | `modules/reservations/domain/value-objects/l2b-room-type-based-charge.value-object.ts` | 0 | DELETE | S — pairs with B2 |
| 94 | `modules/reservations/domain/value-objects/linked-name-stationery.value-object.ts` | 0 | DELETE | S |
| 95 | `modules/reservations/domain/value-objects/membership-rate-rule.value-object.ts` | 0 | DELETE | S — pairs with B8 |
| 96 | `modules/reservations/domain/value-objects/reservation-party.value-object.ts` | 1 | DELETE | S — paired spec `__tests__/party-invariants.spec.ts`; S2 parties asset unused by shipped query; owner confirm |
| 97 | `modules/reservations/domain/value-objects/share-defaults.value-object.ts` | 0 | DELETE | S — live `get-share-defaults.handler` does not use it; owner confirm |
| 98 | `modules/reservations/infrastructure/adapters/prisma-front-office-handoff.adapter.ts` | 0 | **RETAIN** | R — explicitly named in R8/ARCH-042; Phase-6 deliverable |
| 99 | `modules/reservations/infrastructure/adapters/prisma-guest-profile.adapter.ts` | 0 | **RETAIN** | R |
| 100 | `modules/reservations/infrastructure/adapters/prisma-inventory-reservation.adapter.ts` | 1 | **RETAIN** | R — also test-referenced (`reservation-hold-mechanism.postgres.spec.ts:2`) |
| 101 | `modules/reservations/permissions/index.ts` | 0 | DELETE | B (content live: `front-office.controller.ts:11`, specs) |
| 102 | `modules/shared/queue.consumers.ts` | 0 | **RETAIN** | W — t567 pin (`:40,53`) |
| 103 | `platform/configuration/dto/setting-response.dto.ts` | 0 | DELETE | S |
| 104 | `platform/registry/dto/register-activity.dto.ts` | 0 | DELETE | S |
| 105 | `platform/registry/dto/register-command.dto.ts` | 0 | DELETE | S |
| 106 | `platform/registry/dto/register-event.dto.ts` | 0 | DELETE | S |
| 107 | `platform/registry/dto/register-module.dto.ts` | 0 | DELETE | S |
| 108 | `platform/registry/dto/register-query.dto.ts` | 0 | DELETE | S |
| 109 | `platform/registry/dto/register-widget.dto.ts` | 0 | DELETE | S |
| 110 | `platform/registry/dto/register-workflow.dto.ts` | 0 | DELETE | S |

### 4.1 Special rows expanded

- **C81–C83 (RETAIN, wiring gap):** handlers unregistered (filename stem absent from `reservations.module.ts`; no other registration site) while frontend API clients exist. Frontend endpoints `GET /reservations/email-templates` and `GET /reservations/package-exclusions` also have **no backend route** (404). The paired frontend panels have 0 render sites. Owner decides: wire (route + registration) or delete **both sides** at B4.3.
- **C41:** the live walk-in-defaults route proves the handler is a dead parallel path, not a wiring gap → DELETE stands.

## 5. Drift, pins, and risks

### 5.1 82 → 110 drift
The audit recorded 82 at audit time; current scan finds 110. The audit's per-file list was never persisted (only category descriptions in §10.1.4/§13), so exact item-level diff is impossible. Current evidence is authoritative for B4.3; every deletion is re-verified anyway (B4.3 acceptance). Delta is consistent with post-audit additions (S2 shares/party assets, config panels, inventory scaffold DTOs).

### 5.2 Transitive-dependents warning (B4.3)
The scan counts **direct** importers. Files whose only importer is a zero-listed file are NOT in the list (e.g. `get-*.query.ts` files under C81–C83, `crs-front-desk-integration`'s neighbours, procurement's controller/service). **Deletion units must be closed transitively** and re-verified at deletion time.

### 5.3 String-path / test pins found
`t567-async-scoping.spec.ts` path-pins: `common/queue/worker-base.ts`, `modules/shared/queue.consumers.ts`, `widget-refresh.workflow.ts` (and live files `events.consumer`, `wash-scheduler`, `gba-reconciliation`, `widget-refresh.activity`). `ws-n-scan-gates.spec.ts` references `main-worker.ts`. Temporal workflows are loaded via `TEMPORAL_WORKFLOWS_PATH` string — **not import-reachable** (this is why those rows are RETAIN W, not dead). No other spec string-references any of the 110 paths (sweep result: 10 hits, all accounted above or false-positive stems resolving to live files).

### 5.4 Documentation conflicts for owner
1. **AGENTS.md** states RLS runs via `property-scope.guard.ts` **and** `partition-router.middleware.ts` — the middleware is unregistered (C24). Ruling needed: wire it, or delete it and correct AGENTS.md.
2. **ARCH-043** claims purchasing/procurement/housekeeping write the legacy inventory world — `procurement.module.ts` is unregistered (C77). Confirm the write path actually in use before authorizing deletion.

## 6. Acceptance-criterion evidence map

| Criterion (source) | Evidence |
|---|---|
| Ledger drafted (B3.5 row) | This document — classes A, B, C with explicit RETAIN/DELETE per item |
| Importer-count evidence (B3.5 row) | §1 method; §2 counts/render sites; §4 all 110 rows with importer counts; named-member 8/8 verification |
| Owner review queued (B3.5 row) | §7 |
| Deletion NOT performed (Phase-3 exit gate) | Zero file removals; git baseline preserved: 751 pre-existing `git status` entries → **751** after (new file sits inside the already-listed untracked `?? docs/reservations/` entry); 0 stashes |
| Evidence recorded, not a test (B3.5 focused-tests column) | No tests added or modified; verification = read-only scans (§8) |
| Deletion deferred with authorization + re-verification (R8/B4.3) | §7 queue + §5.2 re-verification requirement restated for B4.3 |

## 7. Owner review queue

| # | Decision needed | Rows affected |
|---|---|---|
| Q1 | Wire or delete `partition-router.middleware` + correct AGENTS.md | C24 |
| Q2 | Confirm procurement write path before authorizing subtree deletion | C77 |
| Q3 | Wiring-gap pairs: wire routes/registrations or delete both sides | C81, C82(+B2), C83 |
| Q4 | Recent S2 shares/parties assets: confirm deletion (incl. paired specs) | C79, C96, C97 |
| Q5 | Cashiering batch-cc / daily-routing feature intent (both sides dead) | C31–C33 |
| Q6 | Inventory scaffold DTO/bars deletion approval (legacy-domain caution) | C59–C76 |
| Q7 | Extras: `room-configs` store methods, `property-description` orphan form, unmounted `EmailTemplatesList`/`PackageExclusionsList`/`L2BFeedbackMessagesList` | §3 extras |
| Q8 | Approve full DELETE set for B4.3 execution (with per-item zero-reference re-check + suite green) | all DELETE rows |

## 8. Verification log (commands, observed results)

```text
node <temp>/zero-importer-scan.js
  → 1320 non-test files; 110 zero-non-test-importer; named 8/8 present (post-fix run; test-importer counts live)

node <temp>/spec-string-pin-sweep.js   (reads scan list, scans all 230 *.spec.ts/*.test.ts)
  → 10 hits: t567(worker-base, queue.consumers, widget-refresh.workflow), t564(partition-router import),
    ws-n-scan-gates(main-worker), t521(main.ts comment), fx-currency.spec(import), cancellation-penalty.spec
    (DOMAIN path — false positive for C84), party-invariants.spec(import), reservation-hold-mechanism.spec(import)

rg "from .+activities|reservations/events" apps/web          → 1 edge (ActivitiesTab→FunctionDiary); barrel 0
rg "EditReservationModal|ReservationDetailModal|ReservationToolbar|ReservationWorkspaceNav" apps/web
  → definition-only; render-site rg "<…" → 0 for all except ReservationSidebarNav (workspace-layout:34)
rg route strings for the 9 config endpoints + extras over apps/api/src
  → absent (9/9 + room-configs + property-description); tax-config/sla/status-flow/pbx-config present
rg "GetEmailTemplatesQuery|GetPackageExclusionsQuery|GetL2BConfigQuery" apps/api/src → own dirs only (unwired)
rg "GetWalkInDefaults|walk-in/defaults" → live service path in front-office.controller (:27,:36-39)
git status --porcelain | Measure-Object → 751 entries before and after (new file under existing `?? docs/reservations/` entry); 0 stashes; zero tracked files modified/deleted by B3.5
```

Scan/sweep scripts intentionally left in temp (not committed) — method is fully specified in §1 for reproduction.

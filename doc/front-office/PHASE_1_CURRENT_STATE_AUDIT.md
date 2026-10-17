# XYLO Front Office — Phase 1: Current-State Audit & Baseline

- **Date:** 2026-10-16
- **Phase:** Phase 1 — Current-State Audit & Baseline (**AUDIT ONLY**)
- **Status:** see `PHASE_1_BASELINE_AND_EXIT_GATE.md` for the exit declaration
- **Related deliverables:**
  - `docs/front-office/PHASE_1_GAP_REGISTER.md`
  - `docs/front-office/PHASE_1_BASELINE_AND_EXIT_GATE.md`
- **Pre-existing architecture artifact (referenced, not duplicated):** `docs/enterprise/front-office-frontend-rebuild-architecture-2026-08-02.md`

This document records the state of the Front Office (FO) domain as it exists in the working tree at the time of audit. It makes no recommendations to implement and changes no application code, schema, configuration, or deployment artifacts.

---

## 1. Executive Summary

The Front Office domain is **substantially implemented but not fully hardened or fully wired**. It is split into two layers that were rebuilt in sequence:

1. **Backend** (`apps/api/src/modules/front-office/`): a CQRS module with a large controller surface (54 routes in `front-office.controller.ts`) plus a dedicated staged check-in submodule (`check-in/`, 11 routes). Check-in, check-out, transfer, upgrade, and change-rate have deep domain implementations with strong unit-test coverage.
2. **Frontend** (`apps/web/features/front-office/`): a large React Query + Zustand feature slice that mirrors the Reservations slice, with a Dashboard shell, room rack, inbound panels, day-use workspace, guest-services workspace, and 11 lazy-loaded operation dialogs. It is wired to the FO endpoints through a typed API layer.

The most material findings are:

- **Authorization gap (P1):** 40 of the 54 endpoints on `front-office.controller.ts` carry **no** `@Permissions`/`@Permission` decorator. This includes state-changing operations (walk-in, check-out, reverse check-in, vouchers, day-use, reg-card, traces/messages/service-requests/wake-up/telephone/room-queue). Any authenticated user can reach them.
- **Property-scope relaxation (P2):** both FO controllers declare `@PropertyScope(false)` at class level, disabling property-scope enforcement for every route in the module.
- **Inert / misleading UI (P2/P3):** the Person-Discrepancy reconcile actions are client-only no-ops; walk-in reports "created and checked in" although the backend only creates a `CONFIRMED` reservation; two dialog kinds (`reg-card`, `checkout-history`) and two guest-service panels (`dayuse`, `history`) are registered but have no render branch.
- **Frontend test coverage is thin (P2):** only 3 frontend test files touch FO, and E2E coverage of FO is a single navigation-handoff test.

The FO backend test suite is healthy in the areas it covers; several backend capabilities (walk-in, reverse check-in, discrepancies, sub-resources, registration card, most read/list handlers) have no dedicated tests.

---

## 2. Scope and Exclusions

**In scope**

- FO routes, pages, layouts, components, dialogs, hooks, stores, and API clients (`apps/web`).
- FO backend controllers, services, command/query handlers, domain services, and persistence touchpoints (`apps/api`).
- FO-adjacent surfaces that the FO UI depends on (billing folio/payment, housekeeping traces, reservations extend/reinstate, CRS front-desk facade).
- Existing tests, type checks, and build/lint commands relevant to FO.

**Out of scope (deferred)**

- Cross-module Reservations↔Front Office integration audit (Phase 2).
- Any implementation, refactor, schema change, or configuration change.
- Re-auditing completed Reservations work. Existing Reservations artifacts are referenced where needed (§11).

**Explicit non-actions taken:** no code edits, no schema/DB changes, no dependency installs, no migrations, no commits, no environment changes. (Confirmed in `PHASE_1_BASELINE_AND_EXIT_GATE.md`.)

---

## 3. Method and Evidence Conventions

- Every finding cites a concrete repository path, and line numbers where useful.
- Evidence classes used in the gap register: **direct** (read from source), **test-derived**, **documentation-derived**, **circumstantial**.
- "Implemented / partial / missing / unverified" classifications are based on the presence and wiring of source, not on UI visibility.
- Where a claim could not be verified (e.g. cross-app `apps/mobile` usage), it is marked **UNKNOWN** rather than assumed.

---

## 4. Architecture and Feature Inventory

### 4.1 Frontend entry points

| Surface | Path | Role | Status |
|---|---|---|---|
| Route page | `apps/web/app/(dashboard)/front-office/page.tsx` | `'use client'`; wraps `<Dashboard/>` in `<Suspense>` | Implemented |
| Route layout | `apps/web/app/(dashboard)/front-office/layout.tsx` | wraps children in `<FrontOfficeProvider>` (QueryClient + realtime bridge) | Implemented |
| Feature barrel | `apps/web/features/front-office/index.ts` | public exports | Implemented |

There are **no nested Next.js route segments** under `/front-office`. Navigation is in-page: a tab bar plus URL-query-driven dialogs.

**Tabs** (`features/front-office/constants/front-office.constants.ts:75-83`): `rack`, `arrivals`, `departures`, `overstayed`, `dayuse`, `activity`, `guest-services`.

**Dialog navigation** is driven by URL query params via `features/front-office/hooks/use-front-office-url-sync.ts`: `tab`, `room`, `reservation`, and `dialog` ∈ {`check-in, walk-in, check-out, transfer, upgrade, extend, change-rate, reverse-check-in, reinstate, redeem-voucher, reg-card, checkout-history, post-charge`}.

### 4.2 Feature inventory (frontend)

| Feature area | Primary source | Entry | Wired? |
|---|---|---|---|
| Dashboard shell | `components/dashboard/Dashboard.tsx` | tab shell | Yes |
| KPI header | `components/dashboard/KpiHeader.tsx` | `useDashboard` + inline `/analytics/dashboard` query | Yes |
| Room rack | `components/rack/RoomRack.tsx`, `RoomCard.tsx`, `RackFilter.tsx` | `useRoomGrid` | Yes |
| Room detail drawer | `components/rack/RoomDetailDrawer.tsx` | `useRoomDetail` | Yes |
| Arrivals | `components/dashboard/ArrivalPanel.tsx` | `useArrivals`, `useMarkNoShowMutation` | Yes |
| Due-outs | `components/dashboard/DueOutPanel.tsx` | `useDueOuts` | Yes |
| Overstayed | `components/dashboard/OverstayedPanel.tsx` | `useOverstayed` | Yes |
| Checkout history | `components/dashboard/CheckoutHistoryPanel.tsx` | `useCheckoutHistory(30)` | Yes |
| In-house guests | `components/guest-services/InHouseGuestsPanel.tsx` | `useRoomGrid` | Yes (Call/Email inert, see register) |
| Person discrepancies | `components/dashboard/PersonDiscrepancyTile.tsx` | `useRoomGrid` | **Partial** — resolve is client-only no-op |
| Activity stream | `components/dashboard/ActivityTimeline.tsx`, `ActivityKpiStrip.tsx` | `useActivityStream` (derived) | Yes |
| Check-in (staged) | `components/dialogs/CheckInDialog.tsx`, `hooks/use-check-in-workspace.ts` | `/check-in/*` | Yes |
| Check-out | `components/dialogs/CheckOutDialog.tsx` → `components/CheckoutFlowModal.tsx` | `/front-office/check-out`, `/billing/*` | Yes |
| Walk-in | `components/dialogs/WalkInDialog.tsx` | `useWalkInMutation` → `POST /front-office/walk-in` | **Partial** — no check-in step (see register) |
| Transfer | `components/dialogs/TransferDialog.tsx` | `useTransferMutation` | Yes |
| Upgrade | `components/dialogs/UpgradeDialog.tsx` | `useUpgradeMutation` | Yes |
| Extend | `components/dialogs/ExtendDialog.tsx` | `useExtendMutation` | Yes |
| Change rate | `components/dialogs/ChangeRateDialog.tsx` | `useChangeRateMutation`, `useRateOptions`, `useRateQuote` | Yes |
| Reverse check-in | `components/dialogs/ReverseCheckInDialog.tsx` | `useReverseCheckInMutation` | Yes |
| Reinstate | `components/dialogs/ReinstateDialog.tsx` | reverse-check-in flow | Yes |
| Voucher redeem | `components/dialogs/VoucherRedeemDialog.tsx` | `useRedeemVoucherMutation` | Yes |
| Registration card | API/hooks exist (`useRegCardUrl`, `getRegCardUrl`) | **no render branch** | **Partial/dead** |
| Guest services workspace | `components/guest-services/GuestServicesWorkspace.tsx` + slide-overs | traces/messages/wake-up/service/queue/telephone | Yes (dayuse/history panels inert) |
| Day-use workspace | `components/day-use/DayUseWorkspace.tsx` + `NewDayUseDialog`, `DayUseDrawer` | `useDayUse`, `useCreateDayUseMutation`, `useCloseDayUseMutation` | Yes |
| Room moves dashboard / mobile | `components/RoomMovesDashboard.tsx`, `MobileBackToBackList.tsx`, `hooks/use-room-moves.ts` | `/front-office/room-moves/*`, `/mobile/*` | **Unused in web** (exported, no consumer) |
| Walk-in defaults panel | `components/WalkInDefaultsPanel.tsx` | `useWalkInDefaults` | **Unused in web** |
| Traces inventory panel | `components/TracesInventoryPanel.tsx` | `useTracesConfig` | **Unused in web** |

### 4.3 React Query surface

- **Read hooks** (`hooks/use-front-office-queries.ts`): dashboard, room-grid, room-detail, arrivals, due-outs, overstayed, checkout-history, traces, messages, wake-up-calls, service-requests, room-queue, day-use, vouchers, telephone-messages, reg-card, registration, tax-config, rate-options, rate-quote.
- **Check-in hooks** (`hooks/use-check-in-workspace.ts`): eligibility, flow, room-options, assign-room, verify-payment, verify-compliance, request-key-issuance, finalize, cancel-in-progress.
- **Mutations** (`hooks/use-front-office-mutations.ts`): check-out, walk-in, transfer, upgrade, extend, change-rate, reverse-check-in, redeem-voucher, add/complete trace, mark no-show, send/mark-read message, schedule/complete wake-up, add/update service-request, room-queue add/ready/remove, create/close day-use, add/link voucher, add/mark-read telephone message.
- **Invalidation:** list mutations invalidate property-scoped keys (`hooks/use-front-office-queries.ts:27-34`); the realtime bridge (`providers/front-office-realtime.bridge.ts`) invalidates on `front-office.*` socket events.

### 4.4 Backend module map

- `front-office.controller.ts` — `@Controller('front-office')`, `@PropertyScope(false)` (line 24). 54 routes.
- `check-in/api/check-in.controller.ts` — `@Controller('check-in')`, `@PropertyScope(false)` (line 50). 11 routes, all `@Permissions`-decorated.
- `front-office.service.ts` — thin CQRS facade; delegates extend/reinstate to the Reservations module.
- `crs-integration/crs-front-desk-integration.service.ts` — facade over `FrontOfficeService` (adds CRS restriction evaluation on change-rate, fire-and-forget email on message).
- `application/commands/*`, `application/queries/*` — CQRS handlers registered in the module constructor.
- `check-in/` — staged check-in submodule (eligibility, flow, room options, assign-room, payment, compliance, key-issuance, finalize, cancel-in-progress, release-room, per-guest check-in).
- `domain/*` — aggregates (`front-office-stay.aggregate.ts`, `room.aggregate.ts`), services (room-assignment, transfer, upgrade, stay-extension, walkin-defaults), value objects.
- `infrastructure/*` — Prisma repositories/adapters.
- `services/reg-card.service.ts` — S3-backed registration card.

---

## 5. Frontend-to-Backend Dependency Map

Global API prefix is `api/v1` (`apps/api/src/main.ts:21`). The web client uses `services/api.ts` with base `NEXT_PUBLIC_API_URL || '/api'`.

| Frontend caller | HTTP | Endpoint | Backend handler → downstream |
|---|---|---|---|
| `useDashboard` / `KpiHeader` | GET | `/front-office/dashboard` | `GetDashboardQuery` → `front-office.query.handlers.ts` |
| `useRoomGrid` | GET | `/front-office/room-grid` | `GetRoomGridQuery` |
| `useRoomDetail` | GET | `/front-office/rooms/:roomNumber` | `GetRoomDetailQuery` |
| `useArrivals` | GET | `/front-office/arrivals` | `GetArrivalsQuery` |
| `useDueOuts` | GET | `/front-office/due-outs` | `GetDueOutsQuery` |
| `useOverstayed` | GET | `/front-office/overstayed` | `GetOverstayedQuery` |
| `useCheckoutHistory` | GET | `/front-office/checkout-history?days=` | `GetCheckoutHistoryQuery` |
| `useTraces`/add/complete | GET/POST | `/front-office/traces*` | `CreateTraceCommand`, `CompleteTraceCommand` |
| `useMessages`/send/mark-read | GET/POST/PUT | `/front-office/messages*` | message commands (+ CRS email) |
| `useWakeUpCalls`/schedule/complete | GET/POST | `/front-office/wake-up-calls*` | wake-up commands → `wake_up_calls` |
| `useServiceRequests`/add/update | GET/POST/PUT | `/front-office/service-requests*` | service-request commands |
| `useRoomQueue`/add/ready/remove | GET/POST/DELETE | `/front-office/room-queue*` | queue commands |
| `useDayUse`/create/close | GET/POST | `/front-office/day-use*` | day-use commands |
| `useVouchers`/redeem/apply/link | GET/POST/PATCH | `/front-office/vouchers*` | voucher commands/queries |
| `useTelephoneMessages`/add/mark-read | GET/POST/PUT | `/front-office/telephone-messages*` | telephone commands |
| `useTaxConfig` | GET | `/front-office/tax-config` | `TaxService.getActiveTaxCodes` |
| `useRegistrationData` | GET | `/front-office/registration/:reservationId` | `GetRegistrationDataQuery` |
| `useRegCardUrl` | GET | `/front-office/reg-card/:reservationId` | `GetRegCardUrlQuery` (**no UI consumer**) |
| check-in workspace | GET/POST | `/check-in/*` | staged check-in commands/queries |
| `useCheckOutMutation` | POST | `/front-office/check-out` | `CheckOutCommandHandler` → billing + reservations + rooms |
| `useWalkInMutation` | POST | `/front-office/walk-in` | `WalkInCommandHandler` → `ReservationsService.create` (CONFIRMED) |
| `useTransferMutation` | POST | `/front-office/transfer` | `TransferRoomHandler` |
| `useUpgradeMutation` | POST | `/front-office/:id/upgrade` | `UpgradeRoomHandler` |
| `useExtendMutation` | POST | `/front-office/:id/extend` | **Reservations** `ExtendStayCommand` |
| `useChangeRateMutation` | POST | `/front-office/:id/change-rate` | CRS restrictions → `ChangeRateCodeHandler` |
| `useReverseCheckInMutation` | POST | `/front-office/reverse-checkin` | `ReverseCheckInHandler` |
| (reinstate dialog) | POST | `/front-office/reinstate` | **Reservations** `ReinstateReservationCommand` |
| `useMarkNoShowMutation` | PUT | `/reservations/:id` `{reservation_status:'NO-SHOW'}` | Reservations module |
| `useRateQuote` | GET | `/rates/engine/quote?…` | rates-inventory engine |
| `CheckoutFlowModal` | GET/POST | `/billing/folio/:id`, `/billing/payment`, `/billing/*` | billing/cashiering |

**Duplicated / competing paths** (see gap register): a legacy top-level `components/CheckoutHistoryModal.tsx`, a feature-local `guest-services/CheckoutHistoryModal.tsx`, and `dashboard/CheckoutHistoryPanel.tsx` all exist; a legacy `store/frontOfficeStore.ts` duplicates FO endpoints used only by two orphan components.

---

## 6. Existing Operational Capabilities

Capabilities are listed **only because they exist in code**, with a status. This is an inventory, not a requirement list.

| Capability | Backend | Frontend | Status |
|---|---|---|---|
| Arrivals / due-outs / overstayed / in-house read | `GetArrivals/DueOuts/Overstayed` + room-grid | ArrivalPanel, DueOutPanel, OverstayedPanel, InHouseGuestsPanel | Implemented |
| Check-in (staged) | `check-in/*` (eligibility→assign→payment→compliance→keys→finalize) | CheckInDialog + workspace hooks | Implemented |
| Per-guest check-in (ADR-075) | `check-in-guest.handler.ts` | **no hook/UI caller** | Backend implemented; frontend unwired |
| Check-out (incl. overstay, early-departure refunds, tax auto-post, folio close) | `check-out.handler.ts` | CheckOutDialog → CheckoutFlowModal | Implemented |
| Room assignment / readiness | `RoomAssignmentService`, `room.aggregate`, `rooms` | rack + check-in room options | Implemented |
| Walk-in | `WalkInCommandHandler` (creates `CONFIRMED` reservation) | WalkInDialog | **Partial** (no auto check-in; misleading toast) |
| Reservation lookup | via room-grid / reservations module | rack row click, URL params | Implemented (cross-module, Phase 2) |
| Guest information / reservation details | `GetRoomDetail`, `GetRegistrationData` | RoomDetailDrawer, dialogs | Implemented |
| Folio / payment / outstanding balance | `billing/*` (folios, postings, payments) | CheckoutFlowModal (uses `useReservationFolios`) | Implemented (cross-module) |
| Transfers / upgrades / extend / change-rate | `TransferRoomHandler`, `UpgradeRoomHandler`, Reservations extend, `ChangeRateCodeHandler` | corresponding dialogs | Implemented |
| Reverse check-in / reinstate | `ReverseCheckInHandler`; Reservations reinstate | ReverseCheckInDialog, ReinstateDialog | Implemented |
| Guest-service tasks | sub-resource handlers | guest-services slide-overs/modals | Implemented |
| Day-use accounts | `Create/CloseDayUseAccount` | DayUseWorkspace/Drawer/Dialog | Implemented |
| Vouchers | voucher commands/queries | VoucherRedeemDialog (+ unused create/link) | Partial |
| Registration card | `RegCardService` + queries | none rendered | Backend implemented; frontend dead |
| Person-discrepancy reconciliation | `GetDiscrepancies`, `ResolveDiscrepancy` | PersonDiscrepancyTile (client-only) | **Partial** (frontend no-op) |
| Room-moves dashboard | `RoomMovesDashboardService` + `/room-moves/*` | components exist but unused | Backend implemented; frontend unused |
| Operational exceptions / status transitions | domain aggregates, status history, `reservation_changes` | surfaced in timeline/panels | Implemented |

---

## 7. Test and Verification Baseline

Full command/result detail is in `PHASE_1_BASELINE_AND_EXIT_GATE.md`. Summary:

- **Backend FO unit tests:** 27 test files under `apps/api/src/modules/front-office/**`. Strong coverage for check-in stages, check-out, transfer, upgrade, change-rate, route permissions, command ownership, events. **Execution status:** a scoped run (`npx jest "modules/front-office"`) was attempted but **crashed on Node heap OOM** on the ~4 GB host after one spec passed; the baseline is **NOT VERIFIED** (recorded in the baseline document §5).
- **Backend uncovered:** walk-in, reverse check-in, discrepancies, sub-resource commands, registration card, and most read/list query handlers; infrastructure adapters/repositories; HTTP-layer check-in (no e2e).
- **Frontend FO tests:** only 3 files (`utils/__tests__/tax-computation.test.ts`, `components/dialogs/__tests__/fo-change-rate-failure.test.tsx`, and `components/__tests__/module-switcher.test.tsx`). Dashboard, rack, check-in workspace, dialogs, and all data hooks are untested.
- **E2E:** exactly one FO-relevant Playwright test — navigation handoff from a reservation kebab action into `/front-office?dialog=check-in&reservation=…` (`e2e/kebab-actions.spec.ts`). No FO workflow E2E.
- **Config notes:** API `appspackage.json` sets `ts-jest` `diagnostics:false` and `tsconfig.json` excludes `__tests__`, so broken test imports would not fail typecheck; API `lint` is `echo 'ok'`.

---

## 8. UI/UX Current-State Assessment

Objective observations separated from subjective opportunities.

**Information hierarchy / organization**

- The Dashboard shell is coherent: header + tabs + a KPI header shared by all data tabs (`components/dashboard/Dashboard.tsx:71-164`). Day-use and guest-services are full-width workspace tabs.
- Positive: per-widget `DataSection` (loading/error/empty) means one failing panel does not blank the board (`components/DataSection.tsx` usage across panels).

**Navigation clarity**

- Tab-based navigation works and the URL-sync hook allows deep links. However, the dialog host exposes two dialog kinds (`reg-card`, `checkout-history`) with no rendered content and no callers, and the guest-services store exposes two panels (`dayuse`, `history`) with no host branch — these are latent dead navigation states (FO-P1-006, FO-P1-007).
- ERP module registry lists both `front-office` (ready) and `front-desk` (integration-pending) pointing at the same `/front-office` route (`lib/os/erpModules.ts:56,58`) — a duplicate destination.

**Status visibility**

- Room status, HK status, VIP and stay bars are surfaced on `RoomCard`; arrivals/departures carry status badges. Status is generally label+color, not color-only (positive for accessibility).

**Action placement / availability by status**

- Quick actions are permission-gated (`components/dashboard/QuickActions.tsx`), and dialogs self-gate. Row-level actions exist on arrivals (Queue / No Show / Check In).
- The Person-Discrepancy actions appear actionable ("Reconcile All", "Verify Occupied", "Mark Vacant") but do not persist (FO-P1-003) — a material UX/trust defect.

**Search / filtering / sorting**

- Rack filtering/search/floor/type/VIP is implemented (`RackFilter`, `RoomRack.matchesFilters`). **Bug (FO-P1-015):** the HK-status filter compares against `room.status` instead of `room.housekeepingStatus` (`RoomRack.tsx:18`), so the HK filter cannot work.

**Forms and validation**

- Dialogs use `react-hook-form` + `zod` schemas (`features/front-office/validation/`), with inline field errors — positive.
- Backend: some routes accept untyped `@Body() any` / inline bodies (walk-in, vouchers, resolve-discrepancy, reinstate, apply/link voucher, reg-card), bypassing DTO validation (FO-P1-005).

**Feedback after actions**

- Toast-based feedback is used. **Defect (FO-P1-004):** walk-in reports "Walk-in created and checked in" although the backend only creates a `CONFIRMED` reservation — misleading.
- Change-rate failure handling is tested and avoids server-text leakage (`fo-change-rate-failure.test.tsx`) — positive.

**Empty / loading / failure states**

- Shared skeletons, empty state, and inline error with retry exist (`components/common/LoadingState.tsx`, `EmptyState.tsx`, `ErrorState.tsx`, `DataSection`).
- Rack empty state and per-panel empty text are present.

**Consistency**

- Generally consistent component patterns across panels; however legacy orphan components and a legacy store duplicate FO concepts (FO-P1-011, FO-P1-016).

**Accessibility / responsive (verifiable)**

- Skeletons expose `role="status"`/`aria-label`. Rack grid is responsive (`grid-cols-2 … xl:grid-cols-10`). Icon-only buttons generally carry `title`. Not exhaustively tested.

---

## 9. Findings Summary (full detail in the Gap Register)

| ID | Severity | Classification | One-line |
|---|---|---|---|
| FO-P1-001 | P1 | Confirmed defect | 40/54 FO controller endpoints lack permission decorators (RBAC gap). |
| FO-P1-002 | P2 | Confirmed defect | Class-level `@PropertyScope(false)` disables property-scope enforcement on all FO/check-in routes. |
| FO-P1-003 | P2 | Confirmed defect | Person-Discrepancy reconcile is a client-only no-op; backend resolve endpoint unused. |
| FO-P1-004 | P2 | Confirmed defect | Walk-in does not check in the guest; success toast claims it did. |
| FO-P1-005 | P2 | Confirmed defect | Untyped `@Body() any` / inline bodies bypass validation on several mutating routes. |
| FO-P1-006 | P2 | Partial implementation | `reg-card` and `checkout-history` dialogs registered with no render branch. |
| FO-P1-007 | P3 | Confirmed defect | `dayuse`/`history` guest-service panels have no host branch (silent no-op). |
| FO-P1-008 | P3 | Improvement opportunity | "Transportation" quick-action is permanently disabled. |
| FO-P1-009 | P3 | Confirmed defect | In-house Call/Email actions are malformed (`tel:<name>`, recipient-less `mailto:`). |
| FO-P1-010 | P3 | Confirmed defect | Day-use drawer "Closed at" is hard-coded `—`. |
| FO-P1-011 | P3 | Potential risk | Legacy orphan FO components + dead `frontOfficeStore.ts`. |
| FO-P1-012 | P2 | Potential risk | Thin frontend FO test coverage; backend capability gaps; single E2E nav test. |
| FO-P1-013 | Info | Unverified/dependency | Referenced "locked" FO discovery/architecture audits are missing from the repo. |
| FO-P1-014 | P3 | Confirmed defect | Duplicate ERP module entries map to the same `/front-office` route. |
| FO-P1-015 | P3 | Confirmed defect | Rack HK-status filter compares against the wrong field. |
| FO-P1-016 | Info | Potential risk | Duplicate checkout-history implementations (three variants). |
| FO-P1-017 | Info | Potential risk | Raw `$queryRawUnsafe`/`$executeRawUnsafe` pervasive across the FO module. |

Severity counts: **P1 ×1, P2 ×6, P3 ×7, Info ×3** (17 findings).

---

## 10. Explicit Unknowns and Limitations

- **Runtime behavior not exercised.** No API/database calls were made; findings about guard enforcement, validation rejection, and skipped specs are derived from source reading, not execution.
- **`apps/mobile` not scanned.** The web `mobile/*` FO hooks/components may be consumed by the mobile app; they are recorded "unused in web".
- **Verification is partial.** Whole-repo typecheck/build was not run and the backend FO unit suite was blocked by heap OOM on the ~4 GB host (web FO suite passed). See baseline doc §1–§5.
- **Cross-module Reservations integration** is deferred to Phase 2.
- **Severity of FO-P1-002** depends on intent; the pattern is long-standing and is also flagged in `docs/RESERVATIONS_FORENSIC_AUDIT.md`. It is rated P2 (not P0) because JWT auth still applies and ALS scoping still resolves the caller's own property in most handlers.

---

## 11. Reservations Context (Reference Only — do not re-audit)

Existing artifacts and integration points Phase 2 should inspect (not re-audited here):

- Specs: `docs/audit/Reservations/` (locked specs; `04_BACKEND_ARCHITECTURE.md`, `05_FRONTEND_ARCHITECTURE.md`, `07_DECISIONS.md` ADR-072/073/074, `10_CODEBASE_MAP.md`, `12_API_CONTRACT.md`).
- Roadmap: `docs/enterprise/reservations-roadmap-v2.md`.
- Forensic baseline: `docs/RESERVATIONS_FORENSIC_AUDIT.md`.
- Backend integration points: `apps/api/src/modules/reservations/` (`ReservationsService.create`, `ExtendStayCommand`, `ReinstateReservationCommand`, `IReservationRepository` single-writer methods used by FO handlers).
- Frontend integration points: `apps/web/features/reservations/` — especially `features/reservations/model/reservation-actions.ts` (`buildCheckInHref()` → `/front-office?tab=arrivals&dialog=check-in&reservation=<id>`), the reservations workspace, and `features/reservations/components/pulse/OperationalPulseStrip.tsx`.
- Shared contract: `packages/shared/src/reservation-state-machine.ts`.

Cross-module completeness is **not** concluded here; it is the designated Phase 2 investigation.

---

*End of Phase 1 current-state audit.*

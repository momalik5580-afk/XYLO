# XYLO Current Architecture Audit

> **Document 01 of the XYLO Architecture Investigation Sequence**
> Status: COMPLETE — CURRENT STATE ONLY. No target design, no gap analysis, no repair plan.
> Basis: repository evidence (code, schemas, configs, tests, build artifacts). The codebase is authoritative.

---

## 1. Audit Scope & Method

### 1.1 Scope

Project-wide architectural audit of the entire XYLO monorepo:

- `apps/api` (NestJS), `apps/web` (Next.js), `apps/admin` (Next.js), `apps/mobile` (Expo)
- `packages/db`, `packages/shared`, `packages/ui-core`, `packages/ui-web`, `packages/ui-native`, `packages/config`, `packages/storage`, `packages/notifications`, `packages/tracing`, `packages/analytics`, `packages/cli`
- Root workspace config, `compose.yaml`, `docker/`, CI workflows, `docs/` (read as claims, not as truth)

Explicitly **out of scope for this document**: Reservations redesign, Availability redesign, target architecture, gap analysis, correction planning. Reservations and Availability were inspected only where they are load-bearing evidence of how the *current* system is structured.

### 1.2 Method

Forensic, evidence-first. Conclusions were derived from:

1. **Static inventory** — file/directory counts, module registration matching against `app.module.ts`, test counts, dead-file scans (zero non-test importers).
2. **Resolved import graph** — every relative `from '...'` specifier in `apps/api/src` resolved to an absolute path to build module→module and layer→layer edge counts; `imports:`/`exports:`/`providers:` extracted from all 56 `*.module.ts` files.
3. **Symbol greps** — `CommandBus`, `PrismaService`, `$transaction`, `$queryRawUnsafe`, `APP_GUARD`, `forwardRef`, `@Controller(`, `@Cron(`, `@Processor(`, `@Body()`, `fetch(`, `useQuery`.
4. **Schema parsing** — model/relation/index/unique counts across all three `.prisma` files; `git status` / `git diff` / `git ls-files` for drift and tracking state.
5. **Representative flow tracing** — HTTP → controller → service/CQRS → repository → Prisma → database for reservation create, check-in, check-out, OTA webhook ingress, availability assertion, inventory purchase.
6. **Test and config inspection** — Jest configs, `tsconfig.json` excludes, CI workflows, `compose.yaml` vs `docker/docker-compose.yml`.

### 1.3 Classification vocabulary (used verbatim in §13)

| Code | Meaning |
|---|---|
| **SOUND** | Working as intended; evidence supports keeping the design |
| **ARCHITECTURAL PROBLEM** | Structural defect in boundaries, ownership, layering, or system shape |
| **TECHNICAL DEBT** | Recognised shortcut/duplication that does not by itself break the structure |
| **LEGACY RESIDUE** | Superseded code/data that still exists and still affects behaviour |
| **IMPLEMENTATION BUG** | Incorrect implementation of an otherwise correct design decision |
| **UNCERTAIN / EVIDENCE REQUIRED** | Cannot be settled from repository evidence alone |

### 1.4 What this document does NOT do

It does not fix, refactor, migrate, rename, delete, or redesign anything. No source file, schema, migration, API, or frontend behaviour was altered during this audit. No architecture documents other than this one were created.

---

## 2. Repository / System Architecture Map

### 2.1 Workspace

```
XYLO/  (pnpm 9 workspaces + Turborepo 2, Node >=20.9)
├── apps/
│   ├── api      NestJS 10 — single HTTP API, ~4,300 source files
│   ├── web      Next.js 16 App Router — ERP/front-desk SPA (~871 code files, ~117k lines)
│   ├── admin    Next.js App Router — corporate panel (27 code files, ~4k lines)
│   └── mobile   Expo Router — guest/staff app (18 code files, ~406 lines)
├── packages/
│   ├── db       Prisma: schema.prisma (907 models) + prisma/inventory.prisma (72) + prisma/platform.prisma (39)
│   ├── shared   @xylo/shared — shared types/utils/domain helpers
│   ├── ui-core | ui-web | ui-native   platform UI libraries
│   ├── config | storage | notifications | tracing | analytics | cli
├── compose.yaml          postgres 16, redis 7, temporal ×3   ← documented stack
├── docker/docker-compose.yml  postgres, redis, minio, api, web, admin, otel, tempo, prometheus, grafana, kong  ← divergent stack
└── docs/  adr, audit/Reservations, availability/phase-4..6, enterprise/, specification/, architecture/ (this document)
```

### 2.2 Backend top-level layering (`apps/api/src`)

| Dir | Files | Intended role | Observed role |
|---|---|---|---|
| `main.ts` / `main-worker.ts` / `app.module.ts` / `worker.module.ts` | 4 | composition root | registers 35 feature modules + platform + common + 3 global guards + 3 global interceptors |
| `core/` | 25 | HTTP/auth/tenant primitives | active; but also owns `tax` (feature-ish) and 5 Prisma-reaching services |
| `platform/` | 150 | tenant-independent platform services (identity, permission, audit, registry, settings, configuration, multi-tenancy, temporal, ai) | fully wired, **zero external consumers on its HTTP surface** |
| `modules/` | 37 dirs, 56 module files, 64 controllers, 187 services | feature domains | 24 substantive, 11 thin, 8 stub, 2 unregistered |
| `common/` | 20 subdirs | cross-cutting (cqrs, database, events, outbox, exceptions, validation, authorization, audit…) | heavily duplicated with `platform/`; several dead subdirs |
| `infrastructure/` | 8 subdirs | technical adapters (prisma, redis, bullmq, temporal, email, payment, pbx, s3, push, dev-seed) | contains one reverse dependency into `modules/` |
| `database/` | 4 SQL files | migrations | orphaned — 0 references from any TS file |

Observed layer-edge counts (resolved relative imports, production code only):

```
modules → common          437      common  → infrastructure   12
modules → infrastructure  173      common  → core              5
modules → core            151      core    → infrastructure    5
platform → common          99      infrastructure → core       1
modules → platform         24      infrastructure → modules    2   ← REVERSE
platform → infrastructure   2      platform → core             1   ← REVERSE
platform → modules          0      common → modules            0
core → modules             0
```

### 2.3 Process / runtime topology

- **One HTTP process** (`main.ts`): `setGlobalPrefix('api/v1')`, OTel SDK init, global guards + interceptors, plus a **background Temporal worker started non-blockingly** inside the same process.
- **One optional worker process** (`main-worker.ts` → `worker.module.ts`): Config + Temporal only.
- **Redis**: 3 ioredis clients in `redis.module.ts` (`REDIS_CLIENT`, `REDIS_PUBLISHER`, `REDIS_SUBSCRIBER`) + a separate BullMQ connection stack + per-consumer connections — ≥4 independent connection stacks.
- **Queues**: BullMQ `events`, `inventory`, `notifications`, `sync`, `audit` registered; 6 `@Cron` sites; 2 `@Processor` classes (one never registered).
- **Temporal**: `temporalio/auto-setup` in `compose.yaml` sharing the app Postgres.

---

## 3. Major Domains and Module Boundaries

Domain boundaries below are **derived from code** (routes, controllers, Prisma models touched, module imports), not from folder names.

### 3.1 Substantive domains (own real logic + own real data)

| Domain | Size | Owns (evidenced by) | Layering present |
|---|---|---|---|
| **reservations** | 251 files | `reservations` aggregate, CQRS 62 registrations, 76 KB repository, domain/{aggregates,entities,events,errors,value-objects,services,ports} | ✅ full DDD |
| **front-office** | 121 files | check-in/out, room transfer/upgrade, `check-in/` nested module, 58+11 registrations, 2 repository tokens | ✅ full DDD |
| **group-allotment** | 185 files | group blocks, allotments, pickup cascade, 32 registrations, 3 repositories + unit-of-work | ✅ full DDD (but controllers hit Prisma directly) |
| **inventory** | 282 files | `xylo_inventory.*` (72 models via `@prisma/inventory-client`), 19 registered sub-modules, own Prisma proxy, costing strategies | ✅ own architecture, **isolated from the rest of the backend** |
| **cashiering** | 67 files | folio routing, FX/posting domain services, 23 registrations | ✅ (handlers use Prisma directly) |
| **availability** | 56 files | `availability_assertion_balances/movements`, `reservation_availability_state`, contracts/ports, reconciliation | ✅ full DDD + transactions |
| **command-center** | 93 files | workspace/tab aggregates, widget registry, `@nestjs/cqrs` stack | ✅ but on a **different CQRS stack** |
| **housekeeping** | 21 files | discrepancy repository, 2 registrations | ✅ partial |
| **activities** | 24 files | banquet/venue/event tables via raw SQL | ❌ controller-owns-SQL |
| **purchasing** | 4 files | **legacy** `purchase_requests`/`inventory_*` on main client, 2,777-line service, 146 Prisma call sites | ❌ god service |
| **billing** | 3 files | folio read/post via raw SQL + stub payment gateway | ❌ thin |
| **rates-inventory** | 16 files | rate codes, CRS engine, 6 domain services | ✅ partial |
| **channels** | 6 files | OTA webhook ingress, `channel_availability` | ✅ ingress only; outbound fabricated |

### 3.2 Thin / stub domains (registered, routable, but no real ownership)

- **STUB (8)** — module + controller + service with literal hardcoded dashboard values: `food-beverage`, `spa-wellness`, `hr-payroll`, `finance-night-audit`, `security-compliance`, `it-infrastructure`, `corporate-board`, `loyalty-engine` (tiers), `channels` (metrics). Example: `modules/spa-wellness/spa-wellness.service.ts:26-39` returns a hardcoded therapist array and `revenueToday: 8450`.
- **THIN (11)** — pure Prisma pass-through, no domain/repo/CQRS: `engineering-maintenance` (32-line service), `companies`, `travel-agents`, `properties`, `profiles`, `pbx-config`, `guests`, `room-service`, `notifications`, `pr-workflow`, `workflow-engine`.
- **UNREGISTERED (2)** — `crs-integration`, `procurement`. `crs-integration`'s *service* is nevertheless re-provided by `front-office.module.ts:5,75`, so the directory is dead as a module but live as a code dependency (and would form a cycle if ever registered).

### 3.3 Platform domains

`platform/` owns identity, permission, audit, configuration, settings, registry (module/widget/workflow/command/query/event/feature), multi-tenancy context, AI registries, Temporal metadata. It correctly **does not import any feature module** (0 edges).

### 3.4 Cross-domain edges that define the real topology

```
front-office ──35──> reservations          (deep: domain/services, domain/events,
                                            application/ports, application/services,
                                            permissions/, root reservations.service)
front-office ──7───> rates-inventory
front-office ──6───> billing
front-office ──3───> availability
activities  ──3───> availability
rates-inventory ──> availability ⇄ mutual forwardRef (cycle)
channels    ──2───> reservations
shared(@Global) ──> group-allotment (application layer) ──> availability
pr-workflow / purchasing ──> workflow-engine, notifications
inventory   ── (isolated: 0 inbound, 0 outbound domain edges)
```

---

## 4. Domain / Module Ownership Analysis

### 4.1 Clear ownership (positive)

- **Availability (assertion engine)**: only `availability/application/services/availability-assertion.service.ts` writes `availability_assertion_balances`, `availability_assertion_movements`, `reservation_availability_state` — all on `tx`. Read side is separated into reconciliation/comparator services. Ownership is unambiguous.
- **Inventory (`xylo_inventory`)**: `modules/inventory/**` contains 124 references to `@prisma/inventory-client`, **0** references to legacy `inventory_*` table names, **0** `@prisma/client` imports. Clean single-writer within its own schema.
- **Platform**: no feature-module imports; identity/permission/audit/settings are self-contained.

### 4.2 Ownership violations and duplication

| # | Violation | Evidence |
|---|---|---|
| 1 | **Front-office writes billing's data directly.** Check-out posts `folio_postings`/charges via `$executeRawUnsafe` outside its own transaction; check-out handler also imports `billing.service`. | `front-office/application/commands/check-out/check-out.handler.ts:6, 376, 386, 462, 472` |
| 2 | **Front-office manipulates reservations' internals** at 7 distinct layers instead of going through the reservations module's public surface. | `check-out.handler.ts:8-16`, `upgrade-room.handler.ts:7-23`, `check-in-workflow.service.ts:4-9`, `front-office.module.ts:64`, `front-office.controller.ts:11` |
| 3 | **Two owners for `property`.** `modules/properties` writes `hotels` (main Prisma); `platform/identity/property.service.ts` writes `platformProperty` (platform Prisma). Duplicate DTOs. | `properties.service.ts:13-50`, `platform/identity/property.service.ts:21-37` |
| 4 | **Two owners for `company`.** `modules/companies` vs `platform/identity/company.service.ts`. | both controllers registered |
| 5 | **Two owners for purchase requests.** `modules/inventory/modules/purchase-*` on `InvPurchaseRequest`; `modules/purchasing` + `modules/procurement` on legacy `purchase_requests`. | `purchasing.service.ts:31,106,176…`, `procurement.service.ts:11,21,88` |
| 6 | **Shared module reaches into a feature application layer.** | `modules/shared/events.consumer.ts:9` → `../group-allotment/application/services/reservation-pickup-cascade.service` |
| 7 | **Availability module reaches into reservations internals.** | `availability.module.ts:14-15` → `@/modules/reservations/application/ports/reservation-availability.port`, `.../infrastructure/repositories/reservation-operation-journal` |
| 8 | **`ReservationOperationJournal` is instantiated twice** (provided by both `reservations.module.ts:114` and `availability.module.ts:45`, exported only by the latter) — latent divergence. | both module files |
| 9 | **Activities controllers own SQL.** No repository, no domain; `availability-sales.controller.ts` is 779 lines with 24 raw-SQL statements. | `activities/*.controller.ts` |
| 10 | **`common/audit` writes to the inventory schema** (`inv_audit_log`) from a `common/` service. | `common/audit/audit.service.ts:14` |

### 4.3 Ownership summary

Ownership is **clear inside the rebuilt domains and at the platform layer, and blurred at every seam between them.** The pattern is consistent: wherever two domains must cooperate, the cooperation is implemented as a *direct internal import* rather than through an owned public interface.

---

## 5. Backend Architecture Analysis

### 5.1 Request pipeline (as registered)

```
app.module.ts providers:
  APP_GUARD     JwtAuthGuard                 (core)
  APP_GUARD     PropertyScopeGuard           (core)
  APP_GUARD     MultiTenantGuard             (core)
  APP_INTERCEPTOR TenantInterceptor
  APP_INTERCEPTOR OtelTracingInterceptor
  APP_INTERCEPTOR IdempotencyInterceptor
  + AuthorizationModule (@Global) → APP_GUARD ×5: Roles, Permission, PropertyAccess,
                                      DepartmentAccess, WarehouseAccess
  + PermissionModule     (@Global) → APP_GUARD ×1: Permission (DB-backed)
                                     = 9 global guards total
  + CommonModule         → APP_FILTER AllExceptionsFilter, APP_INTERCEPTOR RequestLogging,
                           APP_INTERCEPTOR ResponseInterceptor
```

### 5.2 CQRS

`common/cqrs/` provides a custom bus: `command-bus.ts` (duplicate-registration guard in `onModuleInit`), `query-bus.ts`, and `pipes.ts` with a 5-pipe chain (Validation → Logging → Authorization → Transaction → Idempotency) wired in `cqrs.module.ts` (`@Global`). Registration is manual per module constructor.

| Module | `register()` calls |
|---|---|
| `reservations.module.ts` | 62 |
| `front-office.module.ts` + `check-in.module.ts` | 58 + 11 |
| `group-allotment.module.ts` | 32 |
| `cashiering.module.ts` | 23 |
| `housekeeping.module.ts` | 2 |
| **total** | **188 across 6 modules** |

**31 of 37 modules use no CQRS at all**, including `inventory` (552 Prisma call sites), `activities` (194), `availability` (32), `purchasing` (146), `billing`, `channels`, `rates-inventory`.

**A second CQRS stack exists**: `command-center` imports `@nestjs/cqrs` (20 files). Both stacks expose tokens literally named `CommandBus`/`QueryBus`; they resolve differently only because `command-center.module.ts` imports the Nest module explicitly.

### 5.3 Layering inside services

- **91 of 187** non-test `*.service.ts` files under `modules/` inject `PrismaService`.
- **6 of 64** controllers inject `PrismaService` directly (worst: `activities/availability-sales.controller.ts`, `command-center.controller.ts:57`).
- **144 files** contain `$queryRawUnsafe`/`$executeRawUnsafe` (production only).
- The **documented shared repository layer is not used**: `BasePrismaRepository` has **2 references repo-wide**, both in `platform/`. Feature modules each define their own interface+impl pair (29 `*.repository.ts` files).

### 5.4 Business-logic placement

- Rebuilt domains: logic lives in domain services / handlers — correct.
- Stub domains: no logic; hardcoded literals returned from services.
- `purchasing.service.ts` — **2,777 lines / 122 KB, 146 Prisma call sites in one class** (god service).
- `activities` — business rules embedded in controllers.
- Frontend-adjacent duplication: `modules/front-office/dto.ts` (192 lines) is a complete parallel reservation DTO set to `modules/reservations/api/dto/*`, both consumed.

### 5.5 Cross-module communication

Three incompatible mechanisms coexist:

1. **Direct internal imports** (78 production module→module edges) — dominant.
2. **In-process event bus** — `common/events/EventBus` → `EventDispatcher`, which has **zero handler registrations** (`register()` is never called outside `event.module.ts`), so `dispatch()` always logs *"No handlers registered for event type"* and returns. The in-process domain-event path is effectively a no-op.
3. **Outbox + BullMQ** — `common/outbox/outbox-processor.ts` (3 `@Cron`s, active) writes/reads `outbox_messages`; `modules/shared/events/outbox.processor.ts` is a **second `OutboxProcessor`** that is constructed and exported by `SharedModule` with **zero callers** (no cron, no lifecycle hook).

### 5.6 Transactions

- Mechanism is well designed: `TransactionPipe` + ambient `AsyncLocalStorage` unit-of-work (`runWithTransaction`, `TransactionManager`, `joinTransaction`).
- **31 production files** use `$transaction`; **43 `joinTransaction` call sites**.
- Erosion: raw SQL passes straight through the tenant proxy and bypasses the ambient transaction; `check-in-guest.handler.ts:49,73,84,100` and `check-out.handler.ts:376,386,462,472` write outside the command transaction; `lost-found`, `activities`, `housekeeping`, `billing` services perform multi-step writes with no transaction at all.

### 5.7 Validation & errors

- **30 of 87** controllers declare `@Body() body: any`.
- `common/exceptions/exception.filter.ts` (`APP_FILTER`) is active; `core/filters/http-exception.filter.ts` (same class name) has 0 importers.
- Two classes named `AppException` exist **in the same directory** (`app-exception.ts` and `app.exception.ts`), both consumed.

---

## 6. Database Architecture Analysis

### 6.1 Schema topology

| File | Models | Datasource | Generator output | Migrations owned |
|---|---:|---|---|---|
| `packages/db/schema.prisma` | **907** | `DATABASE_URL` → `public` | default `@prisma/client` | `packages/db/migrations/` (51 dirs, 24 tracked) |
| `packages/db/prisma/inventory.prisma` | **72** | `INVENTORY_DATABASE_URL`, `schemas=["xylo_inventory"]` | `../node_modules/.prisma/inventory-client` | **none of its own** — inventory DDL folded into main migrations |
| `packages/db/prisma/platform.prisma` | **39** | `PLATFORM_DATABASE_URL`, `?schema=xylo_platform` | `../node_modules/.prisma/platform-client` | **none**; file is **untracked in git** |

**1,018 models over one physical database.** ~639 models carry Prisma `db pull` introspection comments, i.e. the schema was largely *reverse-engineered from a live database*, not authored.

### 6.2 Client wiring

```
apps/api/package.json:
  "@prisma/client": "5.22.0"
  "@prisma/inventory-client": "file:../../packages/db/node_modules/@prisma/inventory-client"
```

Verified drift:

- `packages/db/prisma/inventory.prisma:21,41,108` → `propertyId … @db.Uuid`
- installed client's embedded `schema.prisma:21,41,108` → `propertyId … @db.VarChar(20)`
- `git diff --stat packages/db/prisma/inventory.prisma` = **207 changed lines, uncommitted**
- installed client still contains 4 models deleted from the schema (`InvPurchaseRequestStatusHistory`, `InvPurchaseRequestApproval`, `InvInventoryReservationAllocation`, `inv_outbox_message`)

The API therefore compiles against a **stale, `file:`-linked inventory client** whose type surface no longer matches the schema.

### 6.3 Duplicated state / multiple sources of truth

| Concept | Copies | Evidence |
|---|---|---|
| Reservation | `reservations` (:9579), `reservation_name` (:9344), plus `check_ins`/`check_outs` | dual-write documented in `rates-inventory/domain/services/reservation-name.domain-service.ts` |
| Guest | `guests` (:4935), `guest_profile` (:4859), `reservation_guests` (:9661) | email/phone/nationality/ID duplicated |
| Room state | `rooms`, `room_status`, `availability.no_of_rooms`, `room_inventory.physical_rooms` | 4 locations |
| Inventory | **15 model pairs** `inventory_*` ↔ `Inv*` + `inventory_*_legacy` | both written by disjoint pipelines |
| Availability counters | `availability`, `room_inventory`, `channel_availability`, `channel_inventory`, `availability_assertion_balances`, `group_block_daily_allocations` | ≥6 |
| Channel inventory | `channel_availability` (keyed `hotel_id`) vs `channel_inventory` (keyed `property_id`) — same grain, different tenancy column | `schema.prisma:1814` / `:1872` |

### 6.4 Referential & index integrity

- **468** String-typed id-like columns across **248** models are not part of any `@relation(fields: [...])` — no FK, convention only.
- **92** models hold an id-like String column with **no relation at all** (incl. `availability.room_type_id`, `rooms` with exactly 1 relation, `audit_trail`, `check_ins`).
- **63** models have neither `@@index` nor `@@unique` (incl. `hotels`, `permissions`).
- `folio_charges` is `@@ignore`d with **no `@id` and no `@@unique`** — Prisma's own comment says it "does not contain a valid unique identifier"; 10 monthly partition models are likewise `@@ignore`d.
- Tables written by raw SQL with **no Prisma model**: `folios`, `folio_postings`, `routing_instructions`, `currencies`, `fx_rates`, `property_currency_settings`, `transaction_codes`, `departments`, `tax_groups`, `allotment_pickups`, `reservation_moves`, `person_discrepancies`, …

### 6.5 Multi-tenancy at the data layer

Four parallel mechanisms, of which only two are active:

| # | Mechanism | Status |
|---|---|---|
| 1 | `PropertyScopeGuard` + `MultiTenantGuard` + `TenantInterceptor` (global) | **ACTIVE** |
| 2 | `request-context.ts` ALS + `infrastructure/prisma/prisma.service.ts` tenant proxy (auto `hotel_id` injection, `SKIP_TENANT_MODELS`) — imported by **188 files** | **ACTIVE** |
| 3 | `platform/multi-tenancy/*` (`ContextValidationGuard`, `ContextPropagationInterceptor`) | registered, **never enforced** (0 `APP_GUARD`, 0 `@UseGuards`) |
| 4 | `modules/inventory/.../inventory-prisma.service.ts` — near-identical proxy with a different skip list (`PROPERTY_SCOPED_MODELS`, 15/72 models) | **ACTIVE, parallel to #2** |

Plus dead: `infrastructure/prisma/extensions/tenant-injection.extension.ts` (never called), `core/middleware/partition-router.middleware.ts` (0 importers, `AppModule` has no `configure()`), `core/guards/partition-access.guard.ts` (0 importers).

**RLS is not reproducible from source**: 0 `ALTER TABLE … ENABLE ROW LEVEL SECURITY`, exactly 1 `CREATE POLICY` (`packages/db/migrations/20260607003000_country_scoped_tax_codes/migration.sql:62`), the policy migration referenced by `packages/db/partitions/ddl/rls_policies_reference.md` **does not exist**, and `set_tenant_context()` is never invoked from TypeScript. Yet 375 models carry Prisma's *"contains row level security"* introspection comment, implying the **live** database has RLS that this repository cannot rebuild.

**Header trust**: `tenant.interceptor.ts:16-29` and `property-scope.guard.ts:35` let `x-property-id` override the authenticated user's tenant; `resource-access.guard.ts:29` states "Property check disabled for development"; `multi-tenant.guard.ts:30-33` rejects a mismatched `x-tenant-id` but there is **no equivalent check for `x-property-id`**.

### 6.6 Migration discipline

- 51 migration directories on disk; **24 tracked in git**, **28 untracked** (not gitignored).
- `packages/db/schema.prisma`: **8,793 uncommitted changed lines**; `packages/db/prisma/platform.prisma`: never committed.
- Root `package.json` exposes **both** `db:push` and `db:migrate`.
- CI (`.github/workflows/ci-test.yml:56-68`) runs `prisma validate` on the **default schema only** — no inventory/platform validation, no drift check.
- `apps/api/src/modules/availability/infrastructure/__tests__/ws-n-scan-gates.spec.ts:373-398` **asserts the dirty git state** (`tracked === []`, `lines.length === 28`, schema files modified) — the drift has been frozen into a gate test rather than resolved.
- Runtime DDL: `lost-found.service.ts:21-35` and `housekeeping.service.ts:11-33` run `CREATE TABLE IF NOT EXISTS` + `CREATE INDEX` in `onModuleInit`.

---

## 7. API Architecture Analysis

### 7.1 Route organisation

- Global prefix `app.setGlobalPrefix('api/v1')` with **no `exclude`** (`main.ts:21`).
- 87 controller files: **21** declare `@Controller('api/v1/...')` (all in `platform/**`) → **`/api/v1/api/v1/...`**; 66 declare relative/bare paths; 2 declare no path at all.
- Swagger advertises the doubled path (`.addServer('api/v1')`, `common/swagger/swagger.setup.ts:12`).
- **No consumer anywhere calls any platform route** (0 refs in `apps/web`, `apps/admin`; no spec hits).

### 7.2 Command/query patterns & mutation semantics

- Rebuilt domains use **explicit commands**: `POST /reservations/:id/{cancel,no-show,confirm,guarantee,reinstate,extend,deposits,waitlist/promote}`, `PUT /:id/room-type`, `PUT /:id/rate`, `POST batch/status`. This is the healthiest area of the API.
- Elsewhere the pattern degrades to generic CRUD: `@Put(` in 64 controllers, `@Post(` in 73, `@Patch(` in only 3.
- Cross-domain mutation via internal service import: `channels/webhook-ingress/webhook.service.ts:3,71` calls `ReservationsService.create` while also writing `channel_availability_log` via raw SQL (`:95`).

### 7.3 Validation, authorization, idempotency

- Validation: 30/87 controllers use `@Body() body: any` (billing's charge/payment/refund, activities controllers, group-allotment raw reads).
- Authorization: **two independent permission vocabularies** — `@Permission(` (173 usages, key `platform_permissions`, DB-backed guard) vs `@Permissions(` (45 usages, key `permissions`, static guard). `front-office.controller.ts` uses **both** on the same file.
- Idempotency: `IdempotencyInterceptor` (global) + `IdempotencyPipe` (CQRS) + a third unused `common/events/idempotency.interceptor.ts`.

### 7.4 Consumers

`apps/web` and `apps/admin` are the only consumers. `apps/web` reaches the API through 4 different client wrappers + 6 feature `api/` directories + 1 raw-fetch bypass in `UniversalSearchBar.tsx:48-49`.

---

## 8. Frontend Architecture Analysis

(Architecture only — no UI/UX assessment.)

### 8.1 Structure

| Layer | `apps/web` files |
|---|---:|
| `features/` (new) | 374 |
| `app/` (incl. 21 `_components/` dirs, 132 files) | 238 |
| `components/` (legacy) | 151 |
| `lib/` | 72 |
| `store/` (Zustand) | 20 |
| `services/`, `hooks/`, `types/` | 13 |

**Three competing organizational schemes coexist**: `features/<domain>/`, `components/<domain>/`, `app/(dashboard)/<domain>/_components/`. Inventory has **no** `features/` dir — it is split across `app/(dashboard)/inventory/**` (147), `components/inventory/**` (65), `lib/inventory/**` (~50).

### 8.2 Data access

Four clients in `apps/web` alone:

| Client | Base default | Consumers |
|---|---|---|
| `lib/api/client.ts` `apiClient` | `/api/v1` | 2 non-test files |
| `services/api.ts` `api` | `/api` | 26 files |
| `lib/inventory/api/client.ts` | wraps `services/api`, re-declares `http://localhost:4000/api/v1` for upload/download | 52 files |
| `lib/command-center/api.ts` | wraps `services/api` | 7 files |

Plus 6 feature `api/` directories with their own path strings, 3 `ApiError` classes, 3 `readCookie()` implementations, 3 response-unwrapping blocks, 2 idempotency implementations (only `services/api.ts`'s is wired into `Providers.tsx`).

React Query is the standard in `features/`/`lib` (83 files import, 70 call `useQuery*`), but the legacy layer does not use it: `components/` → 3 files; `app/` pages → 2 files (while **61 app files pull global Zustand**).

### 8.3 State management

29 Zustand stores total (web global 20, web feature 4, admin 2, mobile 3); **no Redux anywhere**.

- **13 of 20** global stores fetch server state (e.g. `housekeepingStore.ts` — 1,402 lines, 28 endpoints; `reservationStore.ts` — 533 lines, 100 API-call sites, 3 consumers).
- **4 orphan stores** with zero consumers: `fnbStore`, `notificationStore`, `settingsStore`, `uiStore`.
- **Concrete duplication**: `store/frontOfficeStore.ts` and `features/front-office/api/front-office.api.ts` fetch the **identical 8 endpoints** (`/front-office/day-use`, `/messages`, `/room-queue`, `/service-requests`, `/telephone-messages`, `/traces`, `/vouchers`, `/wake-up-calls`) into two caches with no shared invalidation.
- **5 query-key registries**; `lib/query-keys.ts` and `features/reservations/hooks/query-keys.ts` use the **same root key `['reservations']` with incompatible shapes**.

### 8.4 Boundaries

- `features → components`: 7 files (front-office dialogs → `LottieMark`, `CheckoutFlowModal`).
- `components → features`: `components/CheckoutFlowModal.tsx:11-16` imports 5 front-office/cashiering internals → **legacy⇄feature cycle**.
- `app/` reaches into feature internals from **19 files** (`(dashboard)/layout.tsx:21,23`, `reporting-analytics/page.tsx:14`, `twin/page.tsx:19-21`).
- Cross-feature production imports: 1 absolute + 8 relative, all reaching into internals (`hooks/`, `lib/`, `components/`) rather than public barrels.
- **No boundary enforcement anywhere**: 0 `no-restricted-imports`, 0 `eslint-plugin-import` boundaries; `apps/web` `lint` script covers only `app components lib` — **`features/` and `store/` (612 of 871 files) are not linted**.

### 8.5 Frontend ↔ backend coupling

- **0 production imports** of `apps/api` or `packages/db` (14 hits inside 3 Jest files that read the source tree).
- `@xylo/shared`: **280** API files vs **10** web files vs **0** admin/mobile/ui. Types are therefore re-declared: `interface Reservation` ×3, `interface GuestProfile` ×3, plus `apps/web/types/reservation.types.ts` (4th file), `features/front-office/dto/*` mirroring the API DTOs, `lib/inventory/api/types.ts` (1,114 lines).
- `apps/admin` and `apps/web` share **no code**; admin re-implements `reservationStore` (462 lines) and drives most dashboards from `generateMockReservations(50)` mock data.
- `apps/mobile`'s axios client (`services/api/client.ts`) has **0 importers**; real calls are raw `fetch` in `hooks/useAuth.ts:7`.

### 8.6 Size/complexity signals

46 `.tsx` files exceed 500 lines; 7 exceed 1,000 (largest: `LinenSection.tsx` 1,572, `AvailabilityPage.tsx` 1,508, `GroupBookingDetailView.tsx` 1,434, `QuickBookForm.tsx` 1,273, `CheckInDialog.tsx` 1,235, `CheckoutFlowModal.tsx` 1,202). Business rules live inside components (checkout totals at `CheckoutFlowModal.tsx:340-407`; FO/HK reconciliation rules at `PersonDiscrepancyTile.tsx:17-30`). `lib/pricing.ts` — the shared pricing module — has **0 consumers**.

---

## 9. Dependency & Layering Analysis

### 9.1 Direction health

| Edge | Count | Verdict |
|---|---:|---|
| `platform → modules` | 0 | ✅ clean |
| `common → modules` | 0 | ✅ clean |
| `core → modules` | 0 | ✅ clean |
| `modules → platform` | 24 | ✅ appropriate (decorators, registries, config) |
| `modules → core` | 151 | ✅ mostly decorators/context/guards |
| `modules → common` | 437 | ✅ (but see §9.3) |
| `infrastructure → modules` | **2** | ❌ `infrastructure/pbx/{pbx.module,pbx.service}.ts` → `modules/pbx-config`; with `SharedModule → PbxModule` this yields `modules → infrastructure → modules` |
| `platform → infrastructure` | 2 | ⚠️ `platform/temporal/{namespace,task-queue}-resolver` |
| `platform → core` | 1 | ⚠️ `tenant-context.service.ts → core/context/request-context` |
| `core → infrastructure` | 5 | ⚠️ auth/tax/interceptors reach `PrismaService` |
| `infrastructure → core` | 1 | ⚠️ `prisma.service.ts → request-context` (required for ALS tenant proxy) |

### 9.2 Circular dependencies

- **Real Nest cycle**: `availability.module.ts:35` ↔ `rates-inventory.module.ts:16` (mutual `forwardRef`).
- **Latent cycle**: `front-office ⇄ crs-integration`, currently masked only because `CrsIntegrationModule` is never registered.
- **Unbalanced `forwardRef`**: `front-office.module.ts:70` forward-refs `ReservationsModule`, which does not import it back.
- **Frontend cycle**: `features/front-office → components/CheckoutFlowModal → features/front-office`.

### 9.3 God modules / god services

| Target | Inbound |
|---|---|
| `reservations` | 3 Nest imports + **41** relative-import edges (35 from front-office), reaching into 7 internal layers |
| `availability` | 6 Nest imports + 8 edges |
| `rates-inventory` | 5 Nest imports + 11 edges |
| `PlatformModule` | 3 Nest imports (clean) |
| `purchasing.service.ts` | 2,777 lines / 146 Prisma call sites in one class |
| `apps/web` god files | 46 components >500 lines; `lib/inventory/hooks/mutations.ts` 1,532 lines; `housekeepingStore.ts` 1,402 lines |

### 9.4 Duplicated implementations (consolidated)

| Concern | Parallel implementations | Status |
|---|---|---|
| Prisma service | **5** (`infrastructure/prisma`, `common/database`, `inventory`, `platform/permission`, `platform/services`); **2 classes both named `PrismaService`**; importers 188 vs 2 | retry wrapper effectively dead |
| CQRS | **2** (custom `common/cqrs`, `@nestjs/cqrs`) with identically named tokens | both active |
| Permission guard | **2** global (`common/authorization`, `platform/permission`), different metadata keys | both active |
| Policy engine | **2** (`common/authorization/{policy.service,policies}`, `platform/permission/policy-evaluation`) | both present |
| Audit | **2 modules + 2 services (both `@Global`, both exporting `AuditService`) + 2 dead interceptors + inventory audit sub-domain + `audit_trail` reader** | mixed |
| Idempotency | **3** (global interceptor active, CQRS pipe active, `common/events` unused) | mixed |
| Exception filter | **2** (`AllExceptionsFilter` ×2, 1 dead); `AppException` ×2 **in the same directory**, both consumed | mixed |
| Registry | **5 families** (`platform/registry`, `platform/temporal`, `platform/services`, `platform/ai`, `command-center/provider-registry`); duplicate `RegisterWorkflowDto`/`RegisterActivityDto`; duplicate `ProviderResolverService` | mixed |
| Auth/identity | **3** (`core/auth` on `users`, `platform/identity` on `platformUser`, `modules/properties` on `hotels`); duplicate property + company CRUD with duplicate DTOs | mixed |
| Event/outbox | **4 paths** (`common/events`, `common/outbox`, `modules/shared/events`, `inventory/infrastructure/outbox`); 2 `OutboxProcessor` classes | mixed |
| Outbox processor | **2** classes named `OutboxProcessor`; shared one wired but never called | mixed |
| Config | **2 × `ConfigModule.forRoot`** in one app, `apps/api/src/config/` **empty**, `@xylo/config` **never imported**, 2 feature-flag systems (env `FEATURE_*` vs DB `platform/configuration`), direct `process.env` in ≥6 places | mixed |
| Frontend HTTP | **4 clients**, 3 `ApiError`, 5 query-key registries, 2 idempotency, 3 `readCookie` | mixed |

---

## 10. Legacy & Technical Debt Analysis

### 10.1 Legacy residue still affecting behaviour

1. **Legacy `availability` counter** — `inventory.domain-service.ts`, documented as *"Single writer for the `availability` counter"*, is an **empty class (38 lines, no methods)**. Non-test scan finds **0** `UPDATE availability SET` / `availability.update` call sites; every `INSERT INTO availability` is in a test fixture. Two reconciliation readers (`availability-reconciliation.service.ts:40`, `counter-balance-comparator.service.ts:39`) still read it. The historical writer was a DB trigger whose DDL file (`scripts/fix_trigger_sold.sql`) is **tracked but deleted from disk**.
2. **Two complete inventory worlds** — `public.inventory_*` (written by `purchasing`, `procurement`, `housekeeping`, `stock/receiving`) coexist with `xylo_inventory.Inv*` (written by `modules/inventory`). 15 model pairs, 2 clients, 2 schemas, **no bridge, no cross-reference table, no reconciliation job**.
3. **`inventory_*_legacy` tables** still present alongside both.
4. **82 files with zero non-test importers**, including all three Reservations Phase-6 port adapters (`prisma-front-office-handoff.adapter.ts`, `prisma-guest-profile.adapter.ts`, `prisma-inventory-reservation.adapter.ts`), the `front-office-stay.aggregate.ts`, 7 CQRS handlers that would 500 if routed, dead guards/middleware/interceptors/filters, and `tenant-injection.extension.ts`.
5. **Dead platform surface** — 6 platform modules (identity, permission, audit, configuration, services, settings) fully registered and guarded by 173 `@Permission` annotations, with **zero consumers and zero tests**.

### 10.2 Technical debt (recognised shortcuts, structure intact)

- 30/87 controllers with `@Body() body: any`.
- 8 stub + 11 thin modules routable but without real logic; fabricated metrics surfaced as live (`channels.service.ts:32-34` → `revenue: bookings * 350`; `finance-night-audit.service.ts:19-24`; `security-compliance.service.ts:13-25`).
- 37 of 55 inventory sub-directories are **empty scaffolds** (zero files).
- Frontend: 46 components >500 lines, business logic in components, 9 hardcoded `localhost` fallbacks, `lib/pricing.ts` unused.
- `apps/admin` mock-driven and parallel; `apps/mobile` effectively a shell (406 lines, empty `components/`, `offline/`).
- Config fragmentation (see §9.4).
- Stub integrations wired as if live: `StubPaymentGateway` hardwired by `createPaymentGateway()`; FCM/APNs are `console.log` stubs with **neither `firebase-admin` nor `@parse/node-apn` installed**; `EmailOutboxWorker` (self-`@deprecated`) polls `email_outbox`, a table **removed by migration `20260706000000`**, disabled only by `EMAIL_OUTBOX_DISABLED` which is absent from `.env.example`.
- Documentation drift: `AGENTS.md` claims Temporal drives the check-in pipeline (`check-in-workflow.service.ts` has **0** temporal references), claims the repository pattern is the convention (`BasePrismaRepository` has 2 consumers), and `apps/api` `lint` is `echo 'ok'`.

### 10.3 Repo hygiene

- 28 tracked-but-deleted files under `packages/db`.
- Orphan SQL `apps/api/src/database/migrations/006..009` with 0 references.
- `packages/shared/src` contains 20 **tracked** `.js`/`.d.ts`/`.map` build artifacts beside `.ts` sources; `apps/mobile/tsconfig.json` remaps `@xylo/shared → src` while web/admin resolve `dist`.
- 40 stale `*.spec.js` in `apps/api/dist` (`deleteOutDir: false`).
- Two divergent compose stacks (§12.9).

---

## 11. Testing Architecture Analysis

### 11.1 Inventory

| Workspace | Test files | Runner | Notes |
|---|---:|---|---|
| `apps/api` | **222** `.spec.ts` | jest (inline config in `package.json`) | 100% colocated in `__tests__/` |
| `apps/web` | **21** `.test.ts` | `next/jest` | 15 of 21 are `readFileSync` source scanners |
| `apps/admin` | **0** | — | no test script, no jest deps |
| `apps/mobile` | **0** | — | no test script |
| all 11 `packages/*` | **0** | — | `@xylo/shared`, `@xylo/db`, storage, tracing, notifications all untested |

### 11.2 Distribution vs domain boundaries

```
group-allotment 75 · reservations 48 · availability 37 · front-office 23
activities 7 · cashiering 6 · rates-inventory 5 · command-center 4
shared 2 · channels 1 · reporting-analytics 1
```

- **88% of module tests sit in 4 rebuild domains.**
- **26 of 37 modules have zero tests**, including **`billing`** (only payment-gateway consumer), **`inventory`** (282 files), `purchasing`, `housekeeping`, `notifications`, `channels` webhook ingress.
- `platform/` (~150 files, 30+ services) has **2 spec files**.
- Tests *are* colocated with modules (structure is domain-aligned), but the mass is skewed.

### 11.3 Quality character

- **79 of 222 (36%)** api specs read source files from disk and assert on code text (conformance/retirement tests), not behaviour.
- **Only 2** specs use `Test.createTestingModule`; **only 2** use `jest.mock`.
- **No e2e/HTTP-level suite exists anywhere.** `pnpm test:e2e` → `jest --config ./test/jest-e2e.json`, and **`apps/api/test/` is an empty directory** — the script cannot run.
- Real-DB integration is genuinely well built: `availability-postgres.harness.ts` creates a disposable schema per run, replays real migration SQL, guards against non-localhost, and `describePostgres` gates **64–65 specs** — but `AVAILABILITY_TEST_DATABASE_URL` is **not set in CI**, so they all `describe.skip` on PRs.

### 11.4 Type-safety blind spot

```json
// apps/api/tsconfig.json:29
"exclude": ["node_modules", "dist", "src/**/__tests__/**"]
// apps/api/package.json jest transform: ["ts-jest", { "diagnostics": false }]
```

- `pnpm typecheck` in `apps/api` **never type-checks any of the 222 test files**.
- Jest transpiles tests with diagnostics off.
- **There is no gate anywhere that type-checks api tests.** A broken import in a test compiles green and fails (or silently skips) at runtime.
- `apps/api/tsconfig.test.json` inherits the same `exclude` and is referenced by no script.
- Consequence already realised: `modules/group-allotment/infrastructure/__tests__/t26-snapshot-consult.spec.ts:2` imports `'../../../domain/ports/pickup-availability.port'`, which resolves to a **non-existent** `src/modules/domain/` — invisible only because of the exclude.
- Asymmetry: `apps/web/tsconfig.json` does **not** exclude tests, so web tests *are* type-checked.

### 11.5 CI reality

`.github/workflows/ci-test.yml`: lint (api's lint is `echo 'ok'` → effectively web-only), typecheck, `pnpm test` (turbo → api+web only), `prisma validate` (default schema only). No DB env, no inventory/platform validation, no migration drift check.

---

## 12. Integration / Operational Architecture Analysis

### 12.1 Temporal — three layers, zero live workflows

| Layer | Contents | Status |
|---|---|---|
| `infrastructure/temporal/` | module, config, client, worker, health, logger; `workflows/index.ts` and `activities/index.ts` are **`export {}`** | wired into `app.module` + `worker.module`; worker loads an **empty workflow barrel** |
| `platform/temporal/` | workflow/activity/worker registries, namespace/task-queue resolvers | `TaskQueueResolverService` has **0 consumers** and resolves queues (`xylo-<tenantId>`) nothing listens on |
| `platform/registry/` | DB-persisted workflow/activity metadata | records metadata, starts nothing |

- Exactly **one workflow exists**: `modules/command-center/infrastructure/temporal/widget-refresh.workflow.ts`.
- `TemporalWorkerService.registerActivities()` / `.registerWorkflow()` have **0 call sites**; no `.workflow.execute/start` anywhere; `TemporalClientService` is consumed only by the health indicator.
- Two worker entry points (API process background + `main-worker.ts`) targeting the same `xylo-default` queue.
- `AGENTS.md` claim "Temporal: durable workflows (check-in pipeline, widget refresh)" — only the widget-refresh *code* exists, and it is never registered.

### 12.2 Queues / cron

- BullMQ queues registered: `events`, `inventory`, `notifications`, `sync`, `audit`.
- `@Cron` ×6: `common/outbox/outbox-processor.ts` (10s/30s/5min), `availability-reconciliation` (hourly), `gba wash-scheduler` (hourly), `gba-reconciliation` (hourly).
- `@Processor` ×2: `modules/shared/events.consumer.ts` (`events`, live) and `queue.consumers.ts` `AnalyticsConsumer` (`analytics`) — the latter is **provided by no module** and its queue is **not registered**.
- `common/queue/queue.factory.ts::createWorker()` never called; `worker-base.ts` never extended.

### 12.3 External integrations

| Integration | Where | Status |
|---|---|---|
| OTA webhooks (Booking.com, Expedia) | `modules/channels/webhook-ingress/` | **live** — HMAC-SHA256 `timingSafeEqual`, dedupe, creates via `ReservationsService` |
| Channels outbound | `channels.service.ts` | **no outbound HTTP**; metrics fabricated |
| CRS | `modules/crs-integration/` | module **never registered** (service re-provided by front-office) |
| Payments | `infrastructure/payment/payment-gateway.ts` | **stub** hardwired (`StubPaymentGateway`); no Stripe/Adyen dependency |
| Email | `infrastructure/email/` | live-if-SMTP-configured; `EmailOutboxWorker` polls a dropped table |
| PBX | `infrastructure/pbx/` | conditional (Asterisk AMI raw TCP + per-hotel HTTP); no tests |
| Push FCM/APNs | `infrastructure/push-gateway/` | **console.log stubs**, SDK packages not installed |
| S3 storage | `packages/storage` + `infrastructure/s3-storage` | wired; `.env` points at `localhost:9000` MinIO, **not present in `compose.yaml`** |

### 12.4 Packages with zero consumers

`@xylo/notifications` (3 zod schemas) — declared in `apps/api/package.json` but **0 imports** repo-wide. `@xylo/config` — presets, 0 importers, and it references a `tsconfig-preset` file that does not exist. `@xylo/ui-native` — all exports are `return null` stubs, **0 consumers**.

### 12.5 Caching

`common/cache` (get/set/del/delPattern/scan/getOrSet/invalidateNamespace + distributed lock) over `REDIS_CLIENT` — consumed by command-center widget cache, configuration cache, permission cache, and the check-in assign-room distributed lock. This is one of the better-wired subsystems. `REDIS_PUBLISHER`/`REDIS_SUBSCRIBER` are exported with **0 consumers**.

### 12.6 Observability

Wired globally and consistently: `initTracing('xylo-api')` in `main.ts:16`, `OtelTracingInterceptor` (APP_INTERCEPTOR), pino `PinoLoggerService` (`@Global LoggerModule`), `RequestLoggingInterceptor`, `AllExceptionsFilter`. Domain spans (`checkInSpan`) used by 11 check-in handlers.

Gaps: `instrumentPrisma` exported but never called; `initTracing` absent from `main-worker.ts`; two parallel logging styles (pino `LoggerModule` ~90 services vs Nest `new Logger(X.name)` in most `modules/*`); the metrics stack (otel-collector/prometheus/tempo/grafana) exists only in the **non-default** `docker/docker-compose.yml`.

### 12.7 Process & worker topology

Workers are not isolated: Temporal worker runs inside the API process by default; `main-worker.ts` boots Config + Temporal only (no queues). BullMQ consumers are reached via `@Processor` classes in `SharedModule`. There is no dedicated queue/worker deployment in either compose stack.

### 12.8 Storage / notifications / analytics

`packages/storage` (S3 + Azure Blob clients, presigned URLs, lifecycle policies) is consumed by `infrastructure/s3-storage`; untested. `packages/analytics` ETL/export are live via `reporting-analytics`; its trigger consumers (`onFolioPost`, `onNightAuditComplete`) are only reachable through the **dead** `AnalyticsConsumer`.

### 12.9 Infrastructure-as-code divergence

| | `compose.yaml` (documented) | `docker/docker-compose.yml` |
|---|---|---|
| services | postgres, redis, temporal, temporal-ui, temporal-admin | postgres, redis, minio, api, web, admin, otel-collector, tempo, prometheus, grafana, kong |
| Temporal | ✅ | ❌ |
| MinIO / OTLP | ❌ | ✅ |

`.env` sets `S3_ENDPOINT=http://localhost:9000` and `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318` — **neither service exists in the documented stack**. Both compose files embed a literal postgres/temporal password fallback.

---

## 13. Architectural Findings Register

Severity is assigned only to **ARCHITECTURAL PROBLEM** rows, per the classification rules.

| ID | Area | Finding | Classification | Severity | Evidence | Architectural Impact | Repairability | Backend Impact |
| -- | ---- | ------- | -------------- | -------- | -------- | -------------------- | ------------- | -------------- |
| ARCH-001 | Structure | pnpm+Turborepo workspace with a single API composition root; app/package split is coherent; `platform`/`common`/`core` do not import feature modules | SOUND | — | `pnpm-workspace.yaml`, `turbo.json`, `app.module.ts`, import-graph (`platform→modules`=0) | Provides a stable skeleton for correction work | n/a | None |
| ARCH-002 | Backend | Rebuilt domains (reservations, front-office, group-allotment, cashiering, housekeeping, availability) implement genuine DDD: aggregates, value objects, domain events, errors, ports, repository interfaces + impls | SOUND | — | `modules/reservations/domain/**`, `front-office/domain/**`, `group-allotment/domain/**`, 29 `*.repository.ts` | The strongest architectural asset; can serve as the reference pattern | n/a | None |
| ARCH-003 | Backend | Custom CQRS bus with 5-pipe pipeline (validation, logging, authorization, transaction, idempotency), duplicate-registration guard, manual per-module registration — 188 handlers across 6 modules | SOUND | — | `common/cqrs/{command-bus,pipes,cqrs.module}.ts`, 5 `*.module.ts` register() sites | Correct command boundary with built-in tx/idempotency | n/a | None |
| ARCH-004 | Database / Testing | Availability assertion engine writes transactionally; 64 DB-gated specs run on disposable schemas with localhost guard and real migration replay | SOUND | — | `availability-assertion.service.ts` (all writes on `tx`), `availability-postgres.harness.ts` | Model for how DB-integrated tests should work repo-wide | n/a | None |
| ARCH-005 | Operations | Observability wired globally (OTel SDK + interceptor, pino, request logging, exception filter) | SOUND | — | `main.ts:16`, `app.module.ts:110`, `common/logging/*` | Cross-cutting concern correctly placed | n/a | None |
| ARCH-006 | Integration | OTA webhook ingress live with HMAC-SHA256 verification, dedupe, and creation through the reservations facade | SOUND | — | `webhook.service.ts:71,95,118`, `channels.module.ts` | Correct inbound adapter pattern | n/a | None |
| ARCH-007 | Security / Tenancy | `x-property-id` header overrides the authenticated user's tenant; property authorization explicitly disabled; no equivalence check (contrast `x-tenant-id`) | ARCHITECTURAL PROBLEM | **Critical** | `core/interceptors/tenant.interceptor.ts:16-29`, `core/guards/property-scope.guard.ts:35`, `common/authorization/resource-access.guard.ts:29`, vs `multi-tenant.guard.ts:30-33` | Tenant isolation is enforced by a client-controllable header, not by authenticated identity — undermines the entire multi-tenant premise | Correctable with controlled migration | Internal backend change only; no API shape change (header semantics change) |
| ARCH-008 | Database / Tenancy | Tenant isolation not reproducible from source: 0 `ENABLE ROW LEVEL SECURITY`, 1 `CREATE POLICY`, referenced RLS migration missing, `set_tenant_context()` never called, Prisma tenant extension dead, `PartitionRouterMiddleware`/`PartitionAccessGuard` never registered — while 375 models introspect as RLS-enabled | ARCHITECTURAL PROBLEM | **Critical** | `packages/db/migrations/**`, `partitions/ddl/rls_policies_reference.md` (points at nonexistent migration), `tenant-injection.extension.ts`, `app.module.ts` (no `configure()`), `partition-router.middleware.ts` (0 importers) | The database's isolation layer cannot be rebuilt, reviewed, or migrated from this repository; behaviour differs between a fresh DB and the live DB | Requires backend structural change + DB evidence first | Database impact; requires coordinated migration |
| ARCH-009 | Database | Three Prisma schemas (907 + 72 + 39 = **1,018 models**) over one physical database, three generators, three datasources; only the main schema has migration ownership | ARCHITECTURAL PROBLEM | High | `packages/db/schema.prisma`, `prisma/inventory.prisma`, `prisma/platform.prisma` | Schema ownership is split while the database is not; cross-schema FKs are impossible (platform↔hotel is a nullable `VarChar(20)`) | Correctable with controlled migration | Database impact; requires coordinated migration |
| ARCH-010 | Database | Migration state unrecoverable from git: 51 dirs / 24 tracked, `schema.prisma` 8,793 uncommitted lines, `platform.prisma` never committed, inventory DDL folded into main migrations, both `db:push` and `db:migrate` exposed, CI validates only the main schema, and a test **asserts** the dirty state | ARCHITECTURAL PROBLEM | High | `git status`, root `package.json`, `ci-test.yml:56-68`, `ws-n-scan-gates.spec.ts:373-398` | No trustworthy baseline for any future schema change; drift is normalised | Correctable with controlled migration | Database impact; requires coordinated migration |
| ARCH-011 | Database / Ownership | Two complete inventory pipelines: 15 `inventory_*` ↔ `Inv*` model pairs, 2 schemas, 2 clients, disjoint writers, no bridge, no reconciliation | ARCHITECTURAL PROBLEM | High | `purchasing.service.ts` (main client), `modules/inventory/**` (inventory client, 124 refs), `housekeeping-*-inventory.service.ts` | Same business process (purchasing/stock) served by two unlinked data worlds; no single source of truth | Correctable with controlled migration | Database impact + API/consumer impact |
| ARCH-012 | Database / Ownership | Dual reservation sources of truth (`reservations` vs `reservation_name`) with documented dual-write; triple guest sources (`guests`, `guest_profile`, `reservation_guests`) | ARCHITECTURAL PROBLEM | High | `schema.prisma:9579/9344/4935/4859/9661`, `reservation-name.domain-service.ts` | Reservation and guest identity can diverge; any domain work must reconcile two models | Correctable with controlled migration | Database impact; internal backend change |
| ARCH-013 | Backend / Dependencies | 78 production cross-module relative-import edges; `front-office → reservations` alone = 35, reaching into 7 internal layers (`domain/entities`, `domain/events`, `domain/services`, `application/ports`, `application/services`, `permissions/`, root service); `reservations` is a god module | ARCHITECTURAL PROBLEM | High | `check-out.handler.ts:6-16`, `upgrade-room.handler.ts:7-23`, `check-in-workflow.service.ts:4-9`, `front-office.controller.ts:11`, `front-office.module.ts:64` | Module boundaries are nominal; any change to reservations internals can break front-office at 35 sites | Correctable with controlled migration | Internal backend change only (no API change) |
| ARCH-014 | Backend / Layering | The documented shared repository layer is unused (`BasePrismaRepository`: 2 refs, both platform); 233 module files reference `PrismaService`; 144 files use raw SQL; 6 controllers inject Prisma directly (incl. 779-line `availability-sales.controller.ts`) | ARCHITECTURAL PROBLEM | High | `common/database/repositories/*`, `activities/*.controller.ts`, `command-center.controller.ts:57` | The stated layering convention is not the actual layering; data access is unbounded | Correctable in place, module by module | Internal backend change only |
| ARCH-015 | Backend / CQRS | Two parallel CQRS implementations with identically named `CommandBus`/`QueryBus` tokens (custom `common/cqrs` vs `@nestjs/cqrs`); 31 of 37 modules use neither | ARCHITECTURAL PROBLEM | High | `common/cqrs/cqrs.module.ts`, `command-center.module.ts:2`, 20 files importing `@nestjs/cqrs` | Ambiguous bus resolution and two command idioms; a wrong import compiles and silently routes to the wrong bus | Correctable with controlled migration | Internal backend change only |
| ARCH-016 | Backend / AuthZ | 9 global `APP_GUARD`s registered across 3 modules, including **two different `PermissionGuard`s** reading different metadata keys, plus two property-access guards; `front-office.controller.ts` uses both `@Permission` and `@Permissions` | ARCHITECTURAL PROBLEM | High | `app.module.ts:107-109`, `authorization.module.ts:18-22`, `permission.module.ts:26`, `front-office.controller.ts` | Authorization semantics depend on which decorator a route happens to use; ordering between 9 enhancers is undocumented and untested | Correctable with controlled migration | API/consumer impact (annotation keys), no URL change |
| ARCH-017 | Backend / Cross-cutting | Pervasive parallel subsystems: 5 Prisma services (2 identically named), 2 `@Global` audit modules exporting the same class name, 2 policy engines, 3 idempotency implementations, 2 exception filters (1 dead), 2 `AppException` in one directory, 5 registry families, 3 auth/identity stacks, duplicate property + company CRUD | ARCHITECTURAL PROBLEM | High | see §9.4 table | Every cross-cutting concern has 2–5 answers; correctness depends on which one a given module happened to import | Correctable with controlled migration | Internal backend change; some API/consumer impact for identity/property CRUD |
| ARCH-018 | Backend / Events | Event delivery fragmented across 4 paths; in-process `EventDispatcher.register()` has **0 callers** so `dispatch()` always no-ops; shared `OutboxProcessor` is constructed and exported with **0 callers**; `AnalyticsConsumer` never registered | ARCHITECTURAL PROBLEM | High | `common/events/event-dispatcher.ts`, `event-bus.ts:29,73`, `shared.module.ts:17-18`, `shared/events/outbox.processor.ts` (no cron/lifecycle) | Domain events are published into a void in-process; cross-module communication is therefore always a direct import | Correctable in place | Internal backend change only |
| ARCH-019 | Backend / Transactions | Transaction boundary erosion: 31 files use `$transaction`, but check-in (`check-in-guest.handler.ts:49,73,84,100`) and check-out (`check-out.handler.ts:376,386,462,472`) post raw SQL outside the command transaction; 4 services write multi-step with no transaction; 144 raw-SQL files bypass the ambient ALS transaction | ARCHITECTURAL PROBLEM | High | §5.6 | Money- and inventory-affecting flows can partially commit; the CQRS `TransactionPipe` guarantee is not universal | Correctable in place (per call site) | Internal backend change only |
| ARCH-020 | Database / Backend | Runtime DDL: `lost-found` and `housekeeping` run `CREATE TABLE IF NOT EXISTS` + `CREATE INDEX` in `onModuleInit` | ARCHITECTURAL PROBLEM | High | `lost-found.service.ts:21-35`, `housekeeping.service.ts:11-33` | Schema evolves outside migration ownership; app boot mutates the database | Correctable with controlled migration | Database impact; requires migration |
| ARCH-021 | Frontend / Data | 4 HTTP clients + 6 feature `api/` dirs in `apps/web`; 3 `ApiError` classes; 5 query-key registries with **incompatible shapes for the same root key**; 2 idempotency implementations (1 unwired); 3 `readCookie()` copies | ARCHITECTURAL PROBLEM | High | §8.2 | No single data-access contract; cache invalidation cannot be reasoned about globally | Correctable with controlled migration | No backend impact |
| ARCH-022 | Frontend / Structure | 3 competing organizational schemes (`features/`, `components/`, `app/**/_components/`); legacy⇄feature import cycle via `CheckoutFlowModal`; 19 `app/` files reach into feature internals; inventory has no `features/` dir | ARCHITECTURAL PROBLEM | High | §8.1, §8.4 | Feature boundaries are unenforceable and partly cyclic | Correctable with controlled migration | No backend impact |
| ARCH-023 | Frontend / State | Server state duplicated: 13 of 20 Zustand stores fetch the API; the identical 8 front-office endpoints are cached in both Zustand and React Query with no shared invalidation; 3 independent `Reservation` type definitions (+4th file); 5 query-key registries | ARCHITECTURAL PROBLEM | High | §8.3 | Stale reads and conflicting caches are structurally possible | Correctable with controlled migration | No backend impact |
| ARCH-024 | Testing | Test architecture blind spots: 26/37 modules untested (incl. `billing`, `inventory`, `purchasing`), 0 e2e (script points at a missing config), 36% of api specs are source-code scanners, only 2 specs boot a Nest module, 0 tests in all packages/admin/mobile, DB suites skipped in CI, platform has 2 specs for ~150 files | ARCHITECTURAL PROBLEM | High | §11 | No behavioural safety net at module or HTTP boundaries; refactors are unguarded | Safe to correct in place (additive) | No backend contract impact |
| ARCH-025 | Backend / Ownership | Front-office check-out posts `folio_postings`/charges directly via raw SQL and imports `billing.service`, while `billing` itself is a 3-file thin module | ARCHITECTURAL PROBLEM | Medium | `check-out.handler.ts:6,376,386,462,472`, `modules/billing/*` | Folio ownership is split; billing cannot enforce its own invariants | Correctable with controlled migration | Internal backend change only |
| ARCH-026 | Backend / Ownership | Parallel entity ownership: `platform/identity` property/company/user CRUD vs `modules/properties` + `modules/companies`, each on a different Prisma client, with duplicate DTOs | ARCHITECTURAL PROBLEM | Medium | `properties.service.ts:13-50`, `platform/identity/property.service.ts:21-37`, duplicate `CreatePropertyDto` | Two writes to "a property" go to two different tables | Correctable with controlled migration | API/consumer impact + database impact |
| ARCH-027 | Backend / Modules | 19 of 37 modules are STUB (8, hardcoded dashboards) or THIN (11, pass-through); 37 of 55 inventory sub-dirs are empty scaffolds; 2 modules never registered | ARCHITECTURAL PROBLEM | Medium | §3.2, `spa-wellness.service.ts:26-39` | Module count overstates system capability; routes exist without owned behaviour | Not currently justified to change (scaffolding) | No backend contract impact |
| ARCH-028 | Backend / Platform | Platform identity/permission/audit/configuration/services/settings surface is fully registered and guarded (173 `@Permission` annotations) but has **zero consumers and zero tests** | ARCHITECTURAL PROBLEM | Medium | 0 refs in `apps/web`/`apps/admin`; 2 platform specs | A large authenticated surface exists that has never been exercised end-to-end | Not currently justified to change | API/consumer impact unknown until consumed |
| ARCH-029 | Backend / Modules | Module cycle `Availability ⇄ RatesInventory` (mutual `forwardRef`); latent `front-office ⇄ crs-integration` cycle masked by an unregistered module; unbalanced `forwardRef` to reservations | ARCHITECTURAL PROBLEM | Medium | `availability.module.ts:35`, `rates-inventory.module.ts:16`, `crs-integration.module.ts:7`, `front-office.module.ts:70` | Registration order is fragile; enabling `crs-integration` would break boot | Correctable with controlled migration | Internal backend change only |
| ARCH-030 | Backend / Layering | Layer violations: `infrastructure/pbx → modules/pbx-config` (creating `modules → infrastructure → modules`); `platform → infrastructure` (2); `core → infrastructure` (5); `SharedModule` (`@Global`) transitively registers `GroupAllotmentModule` + `AvailabilityModule` with a factually stale "no cycle" comment | ARCHITECTURAL PROBLEM | Medium | `infrastructure/pbx/pbx.module.ts:4`, `shared.module.ts:14-16` | Layer rules are already broken at the composition root; global scope leaks feature modules | Correctable with controlled migration | Internal backend change only |
| ARCH-031 | Frontend / Governance | No boundary enforcement anywhere: 0 `no-restricted-imports` / import-boundaries rules; `apps/web` lint covers only `app components lib`, leaving `features/` + `store/` (**612 of 871 files**) unlinted | ARCHITECTURAL PROBLEM | Medium | `apps/web/.eslintrc.json`, `apps/web/package.json` lint script | Frontend boundaries are convention-only and already violated (§8.4) | Safe to correct in place | No backend impact |
| ARCH-032 | Operations / Temporal | Temporal has 3 overlapping layers; exactly 1 workflow exists and is never registered or started; the worker loads an `export {}` workflow barrel; `TaskQueueResolverService` resolves queues nothing listens on; AGENTS.md claims a check-in pipeline workflow | ARCHITECTURAL PROBLEM | Medium | `infrastructure/temporal/workflows/index.ts`, `temporal-worker.service.ts` (0 register call sites), `platform/temporal/*` | Durable-execution infrastructure is provisioned but functionally inert | Not currently justified to change until a real workflow is required | No backend contract impact |
| ARCH-033 | API / Validation | 30 of 87 controllers accept `@Body() body: any`; DTO sets duplicated (`front-office/dto.ts` vs `reservations/api/dto/*`) | TECHNICAL DEBT | Medium | §5.7 | Unvalidated input at the edge; two reservation DTO vocabularies | Safe to correct in place | API/consumer impact possible (stricter validation) |
| ARCH-034 | Frontend | 46 `.tsx` >500 lines (7 >1,000); business logic inside components (checkout totals, FO/HK reconciliation); `lib/pricing.ts` unused | TECHNICAL DEBT | Medium | §8.6 | Component-level coupling; logic untestable in isolation | Safe to correct in place | No backend impact |
| ARCH-035 | Frontend | 9 hardcoded `localhost` fallbacks, 3 `readCookie` copies, 3 response-unwrapping blocks, 2 login paths, web/admin configs duplicated-but-different | TECHNICAL DEBT | Medium | §8.2, §8.5 | Environment-dependent behaviour; duplicated auth logic | Safe to correct in place | No backend impact |
| ARCH-036 | Frontend | `apps/admin` is a mock-driven parallel app (462-line `reservationStore` re-declaring web types, 0 API calls, no React Query); `apps/mobile` is a shell (406 lines, dead axios client, empty `components/`+`offline/`) | TECHNICAL DEBT | Medium | §8.5 | Two frontends neither share code nor integrate with the backend consistently | Not currently justified to change | Unknown / evidence required |
| ARCH-037 | Config | Two `ConfigModule.forRoot` in one app; `apps/api/src/config/` empty; `@xylo/config` never imported; 2 feature-flag systems (env vs DB); direct `process.env` in ≥6 places | TECHNICAL DEBT | Medium | `app.module.ts:62`, `common/config/config.module.ts`, `platform/configuration/feature-flag.service.ts` | Configuration has no single authority | Safe to correct in place | No backend contract impact |
| ARCH-038 | Integration | Stub integrations wired as if live: `StubPaymentGateway` hardwired; FCM/APNs `console.log` with SDKs not installed; `EmailOutboxWorker` polls a table removed by migration `20260706000000`; channel metrics fabricated (`revenue: bookings * 350`) | TECHNICAL DEBT | Medium | `payment-gateway.ts`, `push-gateway/*`, `email-outbox.worker.ts`, `channels.service.ts:32-34` | Operators cannot distinguish live from simulated behaviour | Safe to correct in place | Internal backend change |
| ARCH-039 | Documentation | Documented architecture diverges from code: AGENTS.md claims Temporal drives the check-in pipeline (0 temporal refs), claims repository pattern is the convention (2 consumers), api `lint` is `echo 'ok'` | TECHNICAL DEBT | Medium | `AGENTS.md`, `check-in-workflow.service.ts`, `common/database/repositories/*` | Agents/developers inherit an inaccurate model of the system | Safe to correct in place | None |
| ARCH-040 | Testing / Build | api tests are excluded from `typecheck` **and** ts-jest runs with `diagnostics:false` → no type gate for 222 tests; `tsconfig.test.json` dead; 40 stale `*.spec.js` in `dist`; `pnpm test:e2e` references a missing config | TECHNICAL DEBT | Medium | `tsconfig.json:29`, `package.json` jest block, `apps/api/test/` (empty) | Broken tests compile green; one already has a non-existent import | Safe to correct in place | No backend contract impact |
| ARCH-041 | Database / Availability | Legacy `availability` counter has **no application writer** — the declared single-writer `inventory.domain-service.ts` is an empty class; only test fixtures write it; 2 reconciliation readers still consume it; the historical DB trigger's DDL is tracked-but-deleted | LEGACY RESIDUE | **High** | non-test grep = 0 writers; `inventory.domain-service.ts` (38 lines); `availability-reconciliation.service.ts:40`, `counter-balance-comparator.comparator`/`counter-balance-comparator.service.ts:39` | A dormant counter still read by reconciliation paths can silently disagree with the live assertion engine | Correctable with controlled migration | Database impact |
| ARCH-042 | Backend | 82 files with zero non-test importers — incl. all 3 Reservations Phase-6 port adapters, the front-office aggregate, 7 CQRS handlers that would 500 if routed, dead guards/middleware/interceptors/filters, and the tenant Prisma extension | LEGACY RESIDUE | Medium | zero-importer scan (§10.1) | Dead code is indistinguishable from live wiring; Phase 6 adapters exist but are unconnected | Safe to correct in place | Internal backend change only |
| ARCH-043 | Database | Legacy `inventory_*` tables still live and written (purchasing, procurement, housekeeping) alongside `xylo_inventory.*`; `inventory_*_legacy` tables present | LEGACY RESIDUE | Medium | `purchasing.service.ts`, `procurement.service.ts`, `housekeeping-*-inventory.service.ts` | Two write paths to the same business concepts remain in production | Correctable with controlled migration | Database impact + API/consumer impact |
| ARCH-044 | Database | Tables accessed by raw SQL with no Prisma model (`folios`, `folio_postings`, `currencies`, `fx_rates`, `routing_instructions`, `transaction_codes`, `allotment_pickups`, …); `folio_charges` `@@ignore`d with no PK/unique; 10 partition models `@@ignore`d; partition DDL outside migrate | LEGACY RESIDUE | Medium | `schema.prisma:4254-4288`, `packages/db/partitions/ddl/create_partition.sql`, raw SQL sites | The Prisma schema is not the inventory of the database; critical money tables are outside the ORM | Correctable with controlled migration | Database impact |
| ARCH-045 | Repo hygiene | 28 tracked-but-deleted files under `packages/db`; orphan SQL `apps/api/src/database/migrations/006-009` (0 refs); 20 tracked build artifacts in `packages/shared/src`; 40 stale `*.spec.js` in `dist` | LEGACY RESIDUE | Low | `git status`, `packages/shared/src` | Repository history and on-disk state disagree | Safe to correct in place | None |
| ARCH-046 | Build / Database | Stale inventory Prisma client: `apps/api` depends on a `file:`-linked `packages/db/node_modules/@prisma/inventory-client` whose embedded schema types `property_id` as `VarChar(20)` while `inventory.prisma` says `Uuid` (207-line uncommitted diff); 4 deleted models still present in the installed client | IMPLEMENTATION BUG | **Critical** | `apps/api/package.json:41`, `packages/db/prisma/inventory.prisma:21,41,108` vs installed `schema.prisma:21,41,108`, `git diff --stat` (207 lines) | The API compiles and runs against a type surface that no longer matches the schema — a latent runtime type/DB mismatch for inventory | Safe to correct in place (regenerate/repoint) | Database impact (client regeneration); internal backend only |
| ARCH-047 | Security | `JWT_SECRET` is inlined into the client bundle via `next.config.js` `env:`; hardcoded fallback secrets in `next.config.js:26` and `app/api/auth/route.ts:8` | IMPLEMENTATION BUG | **Critical** | `apps/web/next.config.js:26,34`, `apps/web/app/api/auth/route.ts:8` | Signing secret exposure invalidates token integrity for the web app | Safe to correct in place | No API contract impact; deployment/config impact |
| ARCH-048 | API | 21 of 87 controllers declare `@Controller('api/v1/...')` under `setGlobalPrefix('api/v1')` with no exclude → `/api/v1/api/v1/...`; Swagger advertises the doubled path | IMPLEMENTATION BUG | Medium | `main.ts:21`, 21 platform controllers, `swagger.setup.ts:12` | Any future consumer must hard-code the doubled path or 404 | Safe to correct in place | API/consumer impact (currently 0 consumers) |
| ARCH-049 | Testing | `pnpm test:e2e` → `jest --config ./test/jest-e2e.json` but `apps/api/test/` is empty and the config does not exist | IMPLEMENTATION BUG | Medium | `apps/api/package.json` scripts, `apps/api/test/` (0 entries) | A documented verification command cannot run | Safe to correct in place | No backend contract impact |
| ARCH-050 | Operations | `EmailOutboxWorker` (self-`@deprecated`) polls `email_outbox`, a table removed by migration `20260706000000`; disabled only by `EMAIL_OUTBOX_DISABLED`, which is absent from `.env.example` | IMPLEMENTATION BUG | Medium | `infrastructure/email/email-outbox.worker.ts`, migration `20260706000000`, `.env.example` | On any default environment the worker polls a nonexistent table every 10s | Safe to correct in place | Internal backend change |
| ARCH-051 | Database | 375 models carry Prisma *"contains row level security"* introspection comments implying live RLS, but the repo contains no enabling DDL or policies | UNCERTAIN / EVIDENCE REQUIRED | — | `schema.prisma` introspection comments; 0 `ENABLE ROW LEVEL SECURITY` | Determines whether ARCH-008 is a documentation gap or a real isolation gap | Requires DB evidence first | Unknown |
| ARCH-052 | Database | Actual runtime mutators of legacy `availability` rows (DB triggers/functions) cannot be determined from the repo; `scripts/fix_trigger_sold.sql` is tracked-but-deleted | UNCERTAIN / EVIDENCE REQUIRED | — | `git ls-files` vs disk; absence of app-side writers | Determines whether ARCH-041 is dormant or still triggered at DB level | Requires DB evidence first | Unknown |
| ARCH-053 | Backend | Actual runtime ordering/priority of the 9 global `APP_GUARD`s cannot be established by static analysis | UNCERTAIN / EVIDENCE REQUIRED | — | 3 registration sites; Nest enhancer sorting | Determines whether any route is silently under-guarded | Requires boot-time evidence | Unknown |

**Register totals: 53 findings — 6 SOUND, 26 ARCHITECTURAL PROBLEM, 8 TECHNICAL DEBT, 5 LEGACY RESIDUE, 5 IMPLEMENTATION BUG, 3 UNCERTAIN.**

**Severity of the 26 architectural problems: 2 Critical, 16 High, 8 Medium, 0 Low.**

---

## 14. Repairability Assessment

Repairability applies to **ARCHITECTURAL PROBLEM** findings (significant problems). Implementation bugs and debt are noted where relevant.

### 14.1 By repairability class

| Repairability | Findings | Notes |
|---|---|---|
| **Safe to correct in place** | ARCH-024 (additive tests), ARCH-031 (lint rules), ARCH-042, ARCH-045, ARCH-046, ARCH-047, ARCH-048, ARCH-049, ARCH-050 | No structural dependency change; individually reversible |
| **Correctable with controlled migration** | ARCH-007, ARCH-009, ARCH-010, ARCH-011, ARCH-012, ARCH-013, ARCH-014, ARCH-015, ARCH-016, ARCH-017, ARCH-018, ARCH-019, ARCH-020, ARCH-021, ARCH-022, ARCH-023, ARCH-025, ARCH-026, ARCH-029, ARCH-030, ARCH-041, ARCH-043, ARCH-044 | Each can be addressed domain-by-domain behind existing routes; requires sequencing and a verification gate per item |
| **High-risk correction** | ARCH-008 (RLS/tenancy), ARCH-011 + ARCH-043 (dual inventory), ARCH-012 (dual reservation) | Data-bearing; wrong sequencing corrupts tenant-visible data |
| **Requires backend structural change** | ARCH-008, ARCH-009, ARCH-011 | Schema/tenant model, not code layout |
| **Not currently justified to change** | ARCH-027 (stub modules), ARCH-028 (unconsumed platform surface), ARCH-032 (inert Temporal) | No functional harm today; revisit when those areas are activated |
| **Evidence required first** | ARCH-051, ARCH-052, ARCH-053 | Cannot be designed without DB/boot evidence |

### 14.2 Key repairability observations

1. **The strongest asset is reusable.** ARCH-002/003 (DDD + CQRS pipes) is a *working* reference implementation. Corrections to ARCH-013/014/015 can be made by **extending the existing pattern outward from the rebuilt domains**, not by inventing a new one.
2. **No widespread circular dependency.** Only one real module cycle exists (`Availability ⇄ RatesInventory`, ARCH-029), and it is `forwardRef`-managed. There is no evidence of an unresolvable tangle.
3. **Frontend coupling to backend is HTTP-only** (0 production imports of `apps/api`/`packages/db`), so frontend corrections (ARCH-021/022/023/031) carry **no backend risk**.
4. **The hard problems are data-shaped, not code-shaped**: ARCH-008 (tenancy enforcement), ARCH-009/010 (schema ownership + migration recovery), ARCH-011/012/041/043 (duplicated sources of truth). These are precisely the items requiring controlled migration and DB evidence.
5. **Several critical items are actually cheap**: ARCH-046 (regenerate/repoint the inventory client) and ARCH-047 (stop inlining `JWT_SECRET`) are in-place fixes with no structural consequence — recorded here only so they are not conflated with structural work.

---

## 15. Backend Impact Assessment

| Backend impact | Findings |
|---|---|
| **No backend contract impact** | ARCH-001..006 (sound), ARCH-021, ARCH-022, ARCH-023, ARCH-024, ARCH-031, ARCH-032, ARCH-034, ARCH-035, ARCH-037, ARCH-039, ARCH-040, ARCH-045, ARCH-047 (config only), ARCH-049 |
| **Internal backend change only** | ARCH-013, ARCH-014, ARCH-015, ARCH-018, ARCH-019, ARCH-025, ARCH-029, ARCH-030, ARCH-038, ARCH-042, ARCH-046, ARCH-050 |
| **API / consumer impact** | ARCH-016 (annotation keys), ARCH-017 (identity/property/company CRUD duplication), ARCH-026 (property/company dual ownership), ARCH-028 (platform surface), ARCH-033 (stricter validation), ARCH-048 (route prefix), ARCH-011/043 (inventory endpoints) |
| **Database impact** | ARCH-008, ARCH-009, ARCH-010, ARCH-011, ARCH-012, ARCH-020, ARCH-041, ARCH-043, ARCH-044, ARCH-046 |
| **Requires coordinated migration** | ARCH-008, ARCH-009, ARCH-010, ARCH-011, ARCH-012, ARCH-041, ARCH-043 |
| **Unknown / evidence required** | ARCH-036, ARCH-051, ARCH-052, ARCH-053 |

**Backend preservation feasibility:** High. The API surface (87 controllers, `api/v1` prefix, explicit command routes in reservations/front-office) is stable and mostly correct. The majority of architectural problems are **internal** — module wiring, duplicate subsystems, layer direction, data-access discipline — and can be corrected without changing endpoint contracts. The set requiring coordinated migration is confined to **data ownership** (tenancy enforcement, schema ownership, dual inventory/reservation/guest models), and those are precisely the items that should be sequenced after evidence collection, not attempted speculatively.

---

## 16. Overall Current-Architecture Assessment

### A. Overall architecture health

**Structurally Problematic but Repairable.**

The system is not fundamentally unsound: it has a coherent workspace, a working composition root, and a genuinely well-built core of DDD+CQRS domains that demonstrate the intended architecture is achievable inside this codebase. But the architecture is **not uniform** — it is a *good architecture in the rebuilt core surrounded by an undifferentiated mass* of stubs, duplicated cross-cutting subsystems, direct cross-module imports, and duplicated data ownership. The problems are at the **seams**, not the centre.

### B. Strong architectural areas

1. **Rebuilt domain cores** (reservations, front-office, group-allotment, cashiering, availability, housekeeping): aggregates, value objects, domain events, ports, repository interfaces + impls — ARCH-002.
2. **Custom CQRS pipeline** with validation/logging/authz/transaction/idempotency pipes and duplicate-registration guards — ARCH-003.
3. **Availability assertion engine**: transactional single-writer design with a well-engineered disposable-schema Postgres test harness — ARCH-004.
4. **Global observability**: OTel + pino + request logging + exception filter wired once, globally — ARCH-005.
5. **Live OTA ingress** with real HMAC verification — ARCH-006.
6. **Platform layer direction is clean** (`platform → modules` = 0 edges).
7. **Cache subsystem** (`common/cache` + distributed lock) is consistently consumed.

### C. Architectural failures

1. **Tenant isolation is client-controlled and not reproducible from source** (ARCH-007 Critical, ARCH-008 Critical) — the single most serious structural issue.
2. **Module boundaries are nominal**: 78 cross-module internal imports, front-office reaching 7 layers into reservations (ARCH-013).
3. **Cross-cutting concerns have 2–5 parallel implementations each** (ARCH-015/016/017) — 9 global guards, 2 permission guards, 2 CQRS buses, 5 Prisma services, 2 audit systems.
4. **Data ownership is duplicated** at the schema level: 1,018 models, dual inventory, dual reservation, triple guest, ≥6 availability counters (ARCH-009/011/012/041/043).
5. **Event/inbox architecture is inert**: the in-process dispatcher no-ops, one outbox processor is wired-but-never-called (ARCH-018).
6. **Frontend has no enforced structure**: three organizational schemes, four HTTP clients, duplicated server state, no lint on 70% of files (ARCH-021/022/023/031).
7. **No behavioural safety net**: 26/37 modules untested, no e2e, api tests not type-checked, DB suites skipped in CI (ARCH-024/040/049).

### D. Technical debt versus structural problems

- **Structural (ARCHITECTURAL PROBLEM, 26)**: boundary erosion, duplicate subsystems, data-ownership duplication, tenancy enforcement, inert event path, frontend structure. These change *how the system must be built going forward*.
- **Debt (8) + Bugs (5) + Legacy (5)**: validation gaps, stub modules, hardcoded metrics, config fragmentation, dead files, stale client, route prefix, mock admin app. These change *how trustworthy individual pieces are*, but not the overall shape.
- The **critical ratio**: 4 Critical findings — 2 structural (tenancy: ARCH-007/008) and 2 implementation (ARCH-046 stale client, ARCH-047 secret exposure). Two of the four are one-line-adjacent fixes; the two structural ones need evidence and migration.

### E. Legacy impact

Legacy is **active, not dormant**. The legacy `inventory_*` pipeline is still written by purchasing/procurement/housekeeping (ARCH-043); the legacy `availability` counter still has readers while its declared writer is an empty class (ARCH-041); 82 files including all three Reservations Phase-6 port adapters are unconnected (ARCH-042); runtime DDL and `@@ignore`d money tables sit outside migration ownership (ARCH-020/044). Legacy does not block new work, but it means **any statement about "current behaviour" must name which of the two pipelines is meant.**

### F. Dependency / layering health

**Mostly healthy direction, broken at 5 edges.** `platform`/`common`/`core` never import feature modules; modules import platform/core appropriately. Violations are concentrated: `infrastructure/pbx → modules/pbx-config` (the only true reverse edge), `platform → infrastructure` (2), `core → infrastructure` (5), plus one real `forwardRef` cycle. The larger problem is not *direction* but *depth*: modules import each other's internals rather than their exports (ARCH-013).

### G. Data ownership health

**The weakest area.** Ownership is clear inside `xylo_inventory` and inside the availability assertion engine, and ambiguous almost everywhere else: dual inventory pipelines with no bridge, dual reservation models, triple guest models, ≥6 availability counter locations, front-office writing folio data, two owners each for property/company/purchase-request, and 468 FK-less id columns across 248 models.

### H. API / backend architecture health

**Mixed and above-average where rebuilt.** Explicit command routes with CQRS, pipes for validation/tx/idempotency, and stable `api/v1` URLs are genuinely good. Degrade points: 30/87 unvalidated bodies, two permission vocabularies on the same controllers, 9 unsequenced global guards, 6 controllers with direct Prisma, 21 double-prefixed dead platform routes, and a platform surface that has never been exercised.

### I. Frontend architecture health

**The least governed layer.** React Query adoption in `features/` is correct and modern, but it coexists with 20 global Zustand stores (13 of them fetching server state), four HTTP clients, five query-key registries, three organizational schemes, a legacy⇄feature import cycle, and no lint coverage over 612 of 871 files. `apps/admin` and `apps/mobile` are effectively prototypes. Positive: 0 production coupling to backend source, and `@xylo/ui-web` is genuinely consumed (277 web files).

### J. Repairability

**Realistically repairable.** No unresolvable cycles, no frontend-backend source entanglement, and a proven in-repo reference architecture (ARCH-002/003) to extend. Roughly 13 findings are safe in-place; 23 are correctable with controlled migration; 4 are high-risk and data-bearing; 3 are not currently justified; 3 need evidence.

### K. Backend preservation feasibility

**High.** The HTTP contract, the module composition root, the CQRS pipes, the observability stack, and the rebuilt domain cores can all be preserved. Corrections are predominantly internal wiring and data-ownership sequencing. Nothing in this audit requires rebuilding the backend.

### L. Critical / High architectural risks (prioritised)

| # | Risk | ID |
|---|---|---|
| 1 | Tenant scope taken from a client-controllable header with property authorization disabled | ARCH-007 |
| 2 | Database isolation (RLS/tenancy) unreproducible from source | ARCH-008 |
| 3 | API running on a stale `file:`-linked inventory Prisma client with type drift | ARCH-046 |
| 4 | `JWT_SECRET` inlined into the client bundle | ARCH-047 |
| 5 | Cross-module internal access making reservations a 41-edge god module | ARCH-013 |
| 6 | Dual inventory + dual reservation + triple guest sources of truth | ARCH-011, ARCH-012, ARCH-043 |
| 7 | Transaction boundary erosion in check-in/check-out/money paths | ARCH-019 |
| 8 | Nine unsequenced global guards / two permission vocabularies | ARCH-016 |
| 9 | Inert in-process event path forcing direct cross-module imports | ARCH-018 |
| 10 | No behavioural test safety net over 26 of 37 modules | ARCH-024 |

**Whether any issue must be addressed before future major domain work:** yes — **ARCH-007 and ARCH-008 (tenancy)** and **ARCH-013 (cross-module internal access)** are the three that compound: every new domain written today against a client-trusted tenant header and against direct imports into `reservations` deepens the correction cost. ARCH-046 and ARCH-047 should be handled immediately as they are cheap and unrelated to structural work.

### M. Unknowns requiring additional evidence

See §17.

---

## 17. Open Evidence / Unknowns

| # | Unknown | Why it matters | Evidence needed |
|---|---|---|---|
| 1 | **Is RLS actually enabled in the live database?** 375 models introspect as RLS-enabled but the repo has no enabling DDL (ARCH-051) | Decides whether ARCH-008 is a documentation gap or a live isolation gap | Query `pg_catalog.relrowsecurity` + `pg_policies` on the live DB; export the missing policy migration |
| 2 | **What actually mutates legacy `availability` rows?** No app writer exists; the trigger's DDL file is deleted (ARCH-052) | Decides whether ARCH-041 is dormant residue or a live DB-level writer | `pg_dump` triggers/functions on `availability`; restore `scripts/fix_trigger_sold.sql` from git history |
| 3 | **Runtime ordering of the 9 global `APP_GUARD`s** (ARCH-053) | Determines whether any route is silently under-guarded or double-evaluated | Boot the API and inspect enhancer metadata / Nest enhancer sort order |
| 4 | **Do the platform HTTP routes resolve as documented or doubled?** Static analysis says `/api/v1/api/v1/...` (ARCH-048) | Confirms whether the entire platform surface is unreachable | Single HTTP request against a booted API |
| 5 | **What is the actual live database schema vs `schema.prisma`?** 8,793 uncommitted lines (ARCH-010) | Determines the true migration baseline before any DB work | `prisma migrate diff` / `db pull` into a scratch file and diff |
| 6 | **Is `xylo_inventory.property_id` actually Uuid or VarChar in the live DB?** (ARCH-046) | Determines whether the stale client is causing live runtime errors today | `\d xylo_inventory."InvItem"` |
| 7 | **Frontend consumers of platform endpoints**: static scan says none | Confirms ARCH-028 is truly unconsumed | Runtime network capture / access logs |
| 8 | **Do `apps/admin`/`apps/mobile` have production consumers at all?** Both appear to be prototypes (ARCH-036) | Determines whether they are in scope for correction or preservation | Deployment configuration / route usage evidence |
| 9 | **BullMQ queue consumer liveness at runtime** — 4 of 5 registered queues appear to have no consumer | Determines whether queued work is silently accumulating | Redis `LLEN` on each queue with the API running |
| 10 | **Whether `AVAILABILITY_TEST_DATABASE_URL` is ever set outside a developer machine** | Determines whether the 64 DB specs provide any CI safety | CI/CD environment inventory |

---

*End of Document 01. No source code, database schema, migration, API, business logic, or frontend behaviour was modified in the course of this audit. No other architecture documents were created.*

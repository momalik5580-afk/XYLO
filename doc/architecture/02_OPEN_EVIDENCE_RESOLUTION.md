# Document 02 — Open Evidence Resolution

**Step 02 of the XYLO Project-Wide Architecture Investigation**

Status: COMPLETE
Precedes: `docs/architecture/01_CURRENT_ARCHITECTURE_AUDIT.md` (Document 01, Step 01)
Date of evidence: 2026-10-08

---

## 1. Document Scope & Constraints

This document resolves the ten open evidence items listed in Document 01 §17 ("Open Evidence / Unknowns"). It is an **evidence artifact only**.

In scope:

- Resolving each of the ten unknowns with direct, reproducible evidence (live database catalog queries, framework library source, CI/deployment configuration, repository search).
- Recording for each item: the original finding reference, the question as posed in Document 01, the investigation performed, the evidence found with exact file/config/database references, a result, a confidence level, and the impact on the original finding.

Out of scope (explicitly not done in this step):

- No source code, database schema, migration, API, frontend behaviour, CI configuration, or runtime configuration was created or modified.
- No target architecture, gap analysis, remediation plan, correction sequencing, or design recommendation is produced here.
- Document 01 is **not** re-opened, re-numbered, or reinterpreted. Where evidence changes how a Document 01 finding should be read, the change is recorded in §15 as an impact statement; the Document 01 register itself is left untouched.
- No new architecture findings are added to the register. Anything newly observed that is not one of the ten questions is recorded in §16 under "New Evidence / Out-of-Scope for Step 02".

---

## 2. Method & Evidence Environment

**Environment**

| Item | Value |
|---|---|
| Repository | `C:\Users\Pro\Desktop\XYLO` (pnpm 9 workspaces + Turborepo) |
| Date | 2026-10-08 |
| OS / shell | Windows 11, PowerShell 5.1 |
| Live database | Docker container `xylo-postgres` (PostgreSQL, healthy), database `xylo_cloud` |
| Database access role | `xylo_user` (superuser) via `docker exec xylo-postgres psql -U xylo_user -d xylo_cloud` |
| Live Redis | Docker container `xylo-redis` (read-only inspection via `redis-cli`) |
| Live API | **Not running.** Only `xylo-postgres`, `xylo-redis`, `xylo-temporal`, `xylo-temporal-ui`, `xylo-temporal-admin` were up during this step |
| Framework source | `node_modules\.pnpm\@nestjs+core@10.4.22_...\node_modules\@nestjs\core` (NestJS 10.4.22) |
| Working-tree state | Pre-existing dirty state (787+ entries per Document 01). This step added **no** tracked changes and removed none |

**Method**

1. **Live database catalog queries** — read-only `SELECT` against `pg_catalog` / `information_schema` only. No `INSERT`, `UPDATE`, `DELETE`, `ALTER`, `CREATE`, `DROP`, or `SET` was executed against the database at any point.
2. **Framework source reading** — where Document 01 asked for boot-time evidence, the ordering/concatenation behaviour was instead derived from the exact installed library source that produces it, and the derivation chain is quoted so it can be checked.
3. **Configuration reading** — GitHub Actions workflows, Kubernetes manifests, Dockerfiles, Compose, `.env` / `.env.example`.
4. **Repository search** — `rg` over `apps/`, `packages/`, `docs/`, `.github/`, `infra/`, excluding `node_modules`, `dist`, `.next`.
5. **Git state** — `git ls-files` vs on-disk and `git status --porcelain` for VCS-completeness questions.

**Deliberate deviations from the evidence procedures named in Document 01**

| Document 01 asked for | What was done instead | Why |
|---|---|---|
| E-03: "Boot the API and inspect enhancer metadata" | Derived ordering from NestJS 10.4.22 source (`scanner.js`, `application-config.js`, `guards-context-creator.js`) | Booting the API would have started pollers and publishers against the live shared database (this is recorded as residual uncertainty in §7 and listed in §16) |
| E-04: "Single HTTP request against a booted API" | Derived route construction from NestJS 10.4.22 `route-path-factory.js` | Same reason; the framework concatenation rule is unconditional and is quoted verbatim in §8 |
| E-05: "`prisma migrate diff` / `db pull` into a scratch file and diff" | Complete model→table-set comparison (all 1,018 Prisma models vs all 1,033 live tables) plus migration-history comparison | `prisma db pull` rewrites `schema.prisma` (a repository mutation). Column-level comparison was performed as spot checks only — see §16 |
| E-07: "Runtime network capture / access logs" | Static consumption search + doubled-path search | API not running; no access-log artifact exists in the repository |
| E-09: "`Redis LLEN` on each queue with the API running" | `redis-cli` census with the API **not** running | API was not started; the census is therefore a persisted-state snapshot, and this is flagged as residual uncertainty in §13 |

---

## 3. Result Vocabulary & Confidence Scale

**Result**

| Result | Meaning |
|---|---|
| `RESOLVED` | The question was answered with direct evidence; the answer is stated in the item |
| `NOT RESOLVED` | Evidence was obtained and it contradicts the premise of the question, or the investigation definitively shows the question cannot be answered as posed |
| `UNABLE TO VERIFY` | Evidence could not be obtained in this step for reasons outside the investigation's control |

Where a single item contains a primary question and a secondary sub-question with different outcomes, the item carries one **Result** (for the primary question) and the sub-question's outcome is stated inline in its own labelled line.

**Confidence**

| Level | Meaning |
|---|---|
| `High` | Direct observation of the live system or of the governing source file; reproducible with a single command |
| `Medium-High` | Derived by a short, fully-quoted chain from primary sources; not captured at runtime |
| `Medium` | Inferred from multiple corroborating static sources; one step of inference remains |
| `Low` | Suggestion only |

**RLS outcome categories** (used in §5; these categories are defined here because Document 01 did not define them)

| Cat | Definition |
|---|---|
| **A** | RLS enabled on the object **and** enforced for the application's database role at runtime |
| **B** | RLS enabled on the object in DDL, but **not** enforced for the application's database role (role is owner / `BYPASSRLS` / `SUPERUSER`, and no `FORCE ROW LEVEL SECURITY`) |
| **C** | RLS present in the live database but **not reproducible** from the repository's source/migrations |
| **D** | RLS absent from the object |

---

## 4. Evidence Summary Register

| ID | Original reference | Question (abridged from Document 01 §17) | Result | Confidence | Impact on original finding |
|---|---|---|---|---|---|
| **E-01** | ARCH-051 → ARCH-008 | Is RLS actually enabled in the live database? | `RESOLVED` | High | ARCH-051 resolved; ARCH-008 **retained and re-characterised** (RLS exists, is far larger than source shows, and is inert for this app) |
| **E-02** | ARCH-052 → ARCH-041 | What actually mutates legacy `availability` rows? | `RESOLVED` | High | ARCH-052 resolved; ARCH-041 **re-characterised** (writers exist in DB, but are attached to tables the application no longer writes) |
| **E-03** | ARCH-053, ARCH-016 | Runtime ordering of the 9 global `APP_GUARD`s | `RESOLVED` | Medium-High | ARCH-053 resolved (JWT first — no ordering-driven under-guarding); ARCH-016 retained; three new side-observations recorded in §16 |
| **E-04** | ARCH-048 | Do platform HTTP routes resolve as documented or doubled? | `RESOLVED` | High | ARCH-048 **confirmed**; knock-on effect on `JwtAuthGuard.PUBLIC_PATHS` recorded in §16 |
| **E-05** | ARCH-010, ARCH-009 | Actual live database schema vs `schema.prisma` | `RESOLVED` (table + migration level) | High | ARCH-010 **confirmed and quantified**; column-level comparison remains open (§16) |
| **E-06** | ARCH-046 | Is `xylo_inventory.property_id` Uuid or VarChar in the live DB? | `RESOLVED` | High | ARCH-046 **confirmed**: live = `uuid`, stale client = `VarChar(20)`; "causing runtime errors today" sub-question = `UNABLE TO VERIFY` |
| **E-07** | ARCH-028 | Frontend consumers of platform endpoints | `RESOLVED` (first-party) | High | ARCH-028 **confirmed unconsumed**; external/unknown callers = `UNABLE TO VERIFY` (§16) |
| **E-08** | ARCH-036 | Do `apps/admin` / `apps/mobile` have production consumers? | `RESOLVED` | High | ARCH-036 **confirmed and refined** (admin deployed but 1 API call site; mobile has a client and no deployment path at all) |
| **E-09** | ARCH-032-adjacent, §12 | BullMQ queue consumer liveness at runtime | `RESOLVED` | Medium-High | Queue topology confirmed: 1 of 6 queues ever carried jobs; wiring gaps quantified |
| **E-10** | §11 (testing), ARCH-024 | Is `AVAILABILITY_TEST_DATABASE_URL` ever set outside a developer machine? | `RESOLVED` | High | Document 01's claim **confirmed**: DB-gated suites skip on every PR |

**Totals: 10 items — 10 `RESOLVED`, 0 `NOT RESOLVED`, 0 `UNABLE TO VERIFY` as primary results; 2 sub-questions `UNABLE TO VERIFY` (E-06, E-07), recorded in §16.**

---

## 5. E-01 — Is RLS actually enabled in the live database?

**Evidence ID:** E-01
**Original finding:** ARCH-051 (`UNCERTAIN / EVIDENCE REQUIRED`), which gates ARCH-008 (`ARCHITECTURAL PROBLEM`, Critical)
**Question (Document 01 §17.1):** "Is RLS actually enabled in the live database? 375 models introspect as RLS-enabled but the repo has no enabling DDL (ARCH-051) — Decides whether ARCH-008 is a documentation gap or a live isolation gap."
**Evidence requested:** `pg_catalog.relrowsecurity` + `pg_policies` on the live DB; export the missing policy migration.

### Investigation performed

1. Counted RLS-enabled and RLS-forced tables per schema via `pg_class`.
2. Enumerated every policy in `pg_policies` and classified each by the session setting it reads.
3. Read table ownership (`relowner`) across all three schemas and the attribute flags of the connecting role.
4. Searched the entire repository for any code that sets the policy GUC.
5. Counted RLS-enabled tables that have no policy at all.
6. Counted RLS DDL across every migration directory on disk.

### Evidence found

**1. RLS is enabled on 386 live tables.**

```
docker exec xylo-postgres psql -U xylo_user -d xylo_cloud -t -A -c \
 "select n.nspname, count(*) filter (where c.relrowsecurity), count(*)
    from pg_class c join pg_namespace n on n.oid = c.relnamespace
   where c.relkind='r' and n.nspname in ('public','xylo_inventory','xylo_platform') group by 1;"
```

| Schema | Base tables | RLS enabled | RLS forced (`relforcerowsecurity`) |
|---|---|---|---|
| `public` | 922 | **386** | **0** |
| `xylo_inventory` | 72 | **0** | 0 |
| `xylo_platform` | 39 | **0** | 0 |
| **Total** | **1,033** | **386** | **0** |

Document 01 recorded "375 models introspect as RLS-enabled"; the live count today is **386**. The catalogue has moved since Document 01 was written, in the direction of *more* RLS.

**2. There are 392 policies, all keyed on a session GUC the application never sets.**

```
select coalesce(nullif(substring(qual from 'app\.[a-z_]+'),'other'),'none'), count(*)
  from pg_policies group by 1 order by 2 desc;
```

| GUC referenced in `qual` | Policies |
|---|---|
| `app.hotel_id` | **375** |
| `app.resort_id` | 8 |
| `qual = 'true'` (no restriction) | 9 |
| **Total** | **392** |

Representative policy shape (identical for the 375):

```sql
CREATE POLICY p_availability ON availability
  USING (hotel_id = current_setting('app.hotel_id', true))
```

The nine `qual = 'true'` policies are effectively no-ops and include conspicuously ad-hoc names — `test_policy` on `account_balances`, plus `api_keys_access`, `general_ledger_access`, `login_history_access`, `employee_salary_access`.

**3. The application connects as a superuser that owns every table, and no table forces RLS.**

```
select rolname, rolsuper, rolbypassrls, rolcreaterole from pg_roles
 where rolname in ('xylo_user','postgres');
```

```
xylo_user|t|t|t          -- (postgres role does not exist)
```

- `.env:2` — `DATABASE_URL="postgresql://xylo_user:***@localhost:5432/xylo_cloud?schema=public"`
- All **1,033** tables across `public`, `xylo_inventory`, `xylo_platform` have `relowner = xylo_user` (a single distinct owner).
- **0** tables have `relforcerowsecurity = true`.

PostgreSQL does not apply RLS to a table's owner unless `FORCE ROW LEVEL SECURITY` is set; `BYPASSRLS` and `SUPERUSER` bypass it unconditionally. All three conditions hold simultaneously here.

**4. No application code sets the policy GUC.**

`rg` over `apps/`, `packages/` for `app.hotel_id | app.current_tenant | set_config | set_tenant_context` returns four hits, none of them reachable:

| File | Line | Reachable? |
|---|---|---|
| `packages/db/src/prisma.extension.ts` | 9 — ``SET app.current_tenant = '${tenantId}'`` | No — `tenantExtension` is exported from `packages/db/src/index.ts:3` but **zero importers in `apps/`** |
| `apps/api/src/infrastructure/prisma/extensions/tenant-injection.extension.ts` | 9 — same statement | No — 0 importers (matches Document 01 §6.5) |
| `packages/db/partitions/scripts/set_tenant_context.sql` | 5 — `set_config('app.current_tenant', …)` | No — SQL function, never invoked |
| `packages/db/partitions/scripts/set_tenant_context.sql` | 2 — `CREATE OR REPLACE FUNCTION` | Definition only |

Two independent defects make any of the above useless even if it were wired: **(a)** the setters write `app.current_tenant` while all 375 policies read `app.hotel_id`; **(b)** `current_setting('app.hotel_id', true)` with `missing_ok = true` returns `NULL` when unset, and `hotel_id = NULL` is never true — so an enforced-but-unset environment returns **zero rows** for every scoped table.

**5. Two RLS-enabled tables have no policy at all:** `tax_codes`, `ar_invoices`.

**6. The repository contains almost no RLS DDL.**

- `rg -il "ENABLE ROW LEVEL SECURITY|CREATE POLICY" packages/db/migrations` → **exactly 1 of 51 migration directories** matches: `packages/db/migrations/20260607003000_country_scoped_tax_codes/migration.sql` (the `CREATE POLICY p_tax_codes` at line 62; the file contains **no** `ENABLE ROW LEVEL SECURITY`).
- **Live divergence:** `tax_codes` has `relrowsecurity = true` but **0 policies** — i.e. the one policy this repository does declare **does not exist in the live database**.
- `packages/db/partitions/ddl/rls_policies_reference.md` exists as a reference document; per Document 01 §6.5 it points at a migration that does not exist. No `ENABLE ROW LEVEL SECURITY` statement exists anywhere in the 51 directories.

### Result

**`RESOLVED`**

RLS **is** enabled in the live database — on **386 tables** with **392 policies** — and the repository reproduces essentially none of it (1 policy file, 0 enabling statements, and even that one policy is absent live).

**RLS outcome category: `B` primary (enabled, but not enforced for the application role — owner + `SUPERUSER` + `BYPASSRLS` + 0 `FORCE`), with `C` also holding (not reproducible from source).** It is not `A`, and it is not `D`.

**Confidence:** `High` — every number above comes from a single read-only catalogue query or a quoted file line, reproducible on demand.

### Impact on the original finding

- **ARCH-051 (`UNCERTAIN / EVIDENCE REQUIRED`) → resolved.** The uncertainty is closed. The finding's own text ("implies the live database has RLS that this repository cannot rebuild") is **confirmed**, and the magnitude is larger than recorded (386 tables / 392 policies vs 375 introspected models; 375 policies read `app.hotel_id`).
- **ARCH-008 (`ARCHITECTURAL PROBLEM`, Critical) → retained; its characterisation is sharpened in both directions.**
  - Sharpened *upward*: the database isolation layer is not merely undocumented — it is 392 policies deep and cannot be rebuilt, reviewed, or migrated from this repository.
  - Sharpened *downward*: the isolation layer is **currently inert**, because the only role that ever connects owns every table and carries `SUPERUSER` + `BYPASSRLS` with no `FORCE`. This is a statement about present enforcement, not about intent or about what a correctly-configured deployment would do.
- **ARCH-007 (header-based tenant scoping, Critical) → materially reinforced, not changed.** With RLS inert, client-controllable `x-property-id` / `x-tenant-id` headers are effectively the only tenant boundary in the request path. This is an impact statement only; ARCH-007's own row is untouched.

---

## 6. E-02 — What actually mutates legacy `availability` rows?

**Evidence ID:** E-02
**Original finding:** ARCH-052 (`UNCERTAIN / EVIDENCE REQUIRED`), which gates ARCH-041
**Question (Document 01 §17.2):** "What actually mutates legacy `availability` rows? No app writer exists; the trigger's DDL file is deleted (ARCH-052) — Decides whether ARCH-041 is dormant residue or a live DB-level writer."
**Evidence requested:** `pg_dump` triggers/functions on `availability`; restore `scripts/fix_trigger_sold.sql` from git history.

### Investigation performed

1. Enumerated every trigger on every relation whose definition mentions `availability`.
2. Read the full source of each function those triggers invoke.
3. Searched every caller of those functions (other functions, triggers, `pg_cron`, the repository).
4. Searched the application for any writer to `availability` — Prisma model writes, raw SQL, and scheduled jobs.
5. Measured row counts and last-modified timestamps on the affected tables.
6. Checked `pg_cron` availability.

### Evidence found

**1. Two live DB-level writers exist and their function bodies demonstrably write `availability`.**

```
select c.relname, t.tgname, pg_get_triggerdef(t.oid)
  from pg_trigger t join pg_class c on c.oid = t.tgrelid
 where not t.tgisinternal and pg_get_triggerdef(t.oid) ~ 'availability';
```

| On table | Trigger | When | Function |
|---|---|---|---|
| `public.allotment_pickup` | `trg_update_allotment_pickup` | AFTER INSERT OR DELETE OR UPDATE | `update_allotment_pickup()` |
| `public.allotment` | `trg_block_availability` | AFTER INSERT OR DELETE OR UPDATE | `block_availability_on_allotment()` |
| `public.availability` | `update_availability_updated_at` | BEFORE UPDATE | `update_updated_at_column()` (timestamp only) |
| `public.channel_availability` | `update_channel_availability_updated_at` | BEFORE UPDATE | `update_updated_at_column()` (timestamp only) |
| `public.availability_assertion_movements` | `availability_assertion_movements_append_only` | BEFORE DELETE OR UPDATE | `availability_assertion_movements_reject_mutation()` (append-only guard) |
| `public.reservation_availability_state` | `reservation_availability_state_link_trg` | AFTER INSERT OR UPDATE OF `current_assertion_id`, DEFERRABLE | `reservation_availability_state_link_check()` (constraint) |

Function bodies (read verbatim from `pg_proc.prosrc`):

- `update_allotment_pickup()` — `UPDATE public.availability SET sold_rooms = sold_rooms ± …, hold_rooms = GREATEST(…)` **and** `UPDATE public.allotment_room_types SET sold_rooms = …`.
- `block_availability_on_allotment()` — loops `generate_series(NEW.begin_date, NEW.end_date, '1 day')` and does `INSERT INTO availability (id, hotel_id, date, room_type, alloted, available, …) … ON CONFLICT`-style upsert for each room type.

**2. Not one of these six names appears anywhere in the repository.**

`rg` for `trg_block_availability | trg_update_allotment_pickup | update_allotment_pickup | block_availability_on_allotment | sp_refresh_availability | sp_daily_availability_refresh` across the whole repo (excluding `node_modules`, `dist`, and Document 01 itself) → **0 hits**.

This includes `packages/db/migrations/**` (all 51 directories) and `packages/db/partitions/**`. Document 01's premise — "the trigger's DDL file is deleted" — is therefore correct and is now shown to be stronger than stated: the DDL is not merely deleted from its original path, it is **absent from every tracked and untracked file in the repository**.

**3. The function *source* requested by Document 01 (`scripts/fix_trigger_sold.sql`) could not be restored from history because it was never a repository artifact under that name** — `git` history was not required, since the live `pg_proc.prosrc` supplied the authoritative current text (captured in full above). The deleted-file question is therefore superseded by better evidence.

**4. Two orphan refresh functions exist with no caller anywhere.**

`sp_daily_availability_refresh` and `sp_refresh_availability` are defined in `public`. Cross-checks:
- No other `pg_proc` body references either name.
- No trigger invokes either.
- **`pg_cron` is not installed** — `select … from cron.job` → `ERROR: relation "cron.job" does not exist`. There is therefore no scheduled job of any kind inside this database.
- Zero repository references (see §2 of this item).

**5. There is no application-level writer to `availability`. Exhaustively checked.**

| Check | Result |
|---|---|
| `prisma.availability.create / update / upsert / delete / updateMany / createMany` in `apps/api/src` | **0 hits** |
| Raw `INSERT INTO availability` / `UPDATE availability` / `DELETE FROM availability` in `apps/`, `packages/` (excluding tests) | **0 hits** |
| `availability-reconciliation.service.ts:40` | `this.prisma.availability.findMany({ … select: { available } })` — **read** |
| `counter-balance-comparator.service.ts:39` | `this.prisma.availability.findMany({ … })` — **read** |
| `@Cron(EVERY_HOUR)` `availability-reconciliation.service.ts:58` | read-only reconciliation |
| `restriction-write.repository.ts` (the availability module's only write repository) | writes `close_to_arrival`, `close_to_departure`, `zero_sell_limits`, `room_type_sell_limits`, `minimum_length_of_stay`, `maximum_length_of_stay`, `audit_trail` — **never `availability`** |
| `availability-assertion.service.ts` | writes `availability_assertion_balances`, `availability_assertion_movements`, `reservation_availability_state` — the *new* engine, never `availability` |
| `legacy-population-assignment.service.ts:76` | `tx.reservation_availability_state.createMany` — never `availability` |

**6. But the trigger tables are no longer written by the application.**

```
allotment         = 1 row
allotment_pickup  = 0 rows      <-- trg_update_allotment_pickup sits here
allotment_pickups = 16 rows     <-- different, plural table
group_pickups     = 77 rows     <-- the canonical GBA table
availability      = 360 rows
```

- The current Group Allotment pipeline persists to `allotment_contracts`, `allotment_daily_quotas`, `allotment_vouchers`, `allotment_stop_sales` and `group_pickups` (`allotment.repository.ts:154` `tx.allotment_contracts.upsert`, `:264/:300/:347` `INSERT INTO allotment_daily_quotas`, `reservation-pickup-cascade.service.ts:113` `UPDATE allotment_pickups`).
- Zero Prisma writes to `allotment` or `allotment_pickup` exist anywhere (`rg "\.allotment\.[a-z]+\(|\.allotment_pickup\.[a-z]+\("` → 0 hits).
- The **only** statements in the entire repository that write `allotment` / `allotment_pickup` are in a DB-gated test: `apps/api/src/modules/group-allotment/infrastructure/__tests__/t65-legacy-freeze.postgres.spec.ts:125,127` (`INSERT INTO allotment …`, `INSERT INTO allotment_pickup …`).
- Result: `trg_update_allotment_pickup` currently has an empty source table; `trg_block_availability` has a single legacy row.

**7. `availability` last changed 2026-09-17.**

`select count(*), max(updated_at) from public.availability` → `360 | 2026-09-17 11:17:14.122766+00`
All 360 rows belong to `hotel_id = 'HRG'`, stay dates `2026-07-31 … 2026-09-28`. Columns: `id, hotel_id, date, room_type, room_type_id, alloted, reserved, available, oversell, waitlist, no_of_rooms, inserted_at, updated_at`. The table also carries `relrowsecurity = true`, `relforcerowsecurity = false`, owner `xylo_user`.

Attribution of that 2026-09-17 write to a specific actor could not be determined: the API was not running during this step, there is no database audit trail on `availability` beyond the `updated_at` trigger, and application logs are not retained in the repository. This sub-item is recorded in §16.

### Result

**`RESOLVED`**

The mutators are two **database-resident** triggers — `trg_update_allotment_pickup` and `trg_block_availability` — plus the `update_availability_updated_at` timestamp trigger. They are provably absent from the repository and provably present in the live database. There is **no application-level writer**, and **no scheduled writer** (`pg_cron` absent).

**Confidence:** `High` for items 1–6 (direct catalogue and source reads, exhaustive negative searches). Attribution of the 2026-09-17 row update: `UNABLE TO VERIFY` (§16).

### Impact on the original finding

- **ARCH-052 (`UNCERTAIN / EVIDENCE REQUIRED`) → resolved.** The unknown is answered: the mutators are the two named triggers/functions, whose current source is reproduced in this document.
- **ARCH-041 → re-characterised; classification moves away from "live DB-level writer" and toward "dormant, structurally live residue".**
  - Document 01's own decision point was "dormant residue **or** a live DB-level writer". The evidence does not select either pole cleanly, and the tie-break is *table activity*, not *object existence*:
    - The trigger objects are live and would fire on any write.
    - The application no longer writes to the trigger tables: `allotment_pickup` has **0 rows**, `allotment` has **1**.
    - The active pipeline writes to different tables (`allotment_contracts`, `group_pickups`, `allotment_daily_quotas`).
  - Therefore ARCH-041's **dual-writer risk is not currently exercised**, but the coupling is **not severed**: a single write to the legacy `allotment` row would silently mutate `availability.alloted`/`available` outside the new assertion engine, and one DB-gated spec (`t65-legacy-freeze.postgres.spec.ts:125,127`) already performs exactly such writes.
  - The correct reading of ARCH-041 is now: *the legacy counter has no active writer from the application, has two active writers from the database, and those database writers are attached to tables the application has all-but abandoned.*
- No new finding is registered for the DB-resident triggers; that observation is recorded in §16 as new evidence.

---

## 7. E-03 — Runtime ordering of the 9 global `APP_GUARD`s

**Evidence ID:** E-03
**Original finding:** ARCH-053 (`UNCERTAIN / EVIDENCE REQUIRED`); interacts with ARCH-016 ("nine unsequenced global guards / two permission vocabularies")
**Question (Document 01 §17.3):** "Runtime ordering of the 9 global `APP_GUARD`s (ARCH-053) — Determines whether any route is silently under-guarded or double-evaluated."
**Evidence requested:** Boot the API and inspect enhancer metadata / Nest enhancer sort order.

### Investigation performed

1. Enumerated every `APP_GUARD` registration site and its module.
2. Enumerated every controller-level `@UseGuards` site.
3. Read the installed NestJS 10.4.22 source that (a) collects global guards during bootstrap and (b) orders them at request time.
4. Read each guard's `canActivate` to determine what it depends on from earlier guards.
5. Counted the metadata decorators each guard reads, to establish double-evaluation.
6. Read `main.ts` for the public-path mechanism.

### Evidence found

**1. The nine registrations, by file.**

| # | Guard class | Registration site |
|---|---|---|
| 1 | `JwtAuthGuard` | `apps/api/src/app.module.ts:107` |
| 2 | `PropertyScopeGuard` | `apps/api/src/app.module.ts:108` |
| 3 | `MultiTenantGuard` | `apps/api/src/app.module.ts:109` |
| 4 | `RolesGuard` | `apps/api/src/common/authorization/authorization.module.ts:18` |
| 5 | `PermissionGuard` (common) | `apps/api/src/common/authorization/authorization.module.ts:19` |
| 6 | `PropertyAccessGuard` | `apps/api/src/common/authorization/authorization.module.ts:20` |
| 7 | `DepartmentAccessGuard` | `apps/api/src/common/authorization/authorization.module.ts:21` |
| 8 | `WarehouseAccessGuard` | `apps/api/src/common/authorization/authorization.module.ts:22` |
| 9 | `PermissionGuard` (platform) | `apps/api/src/platform/permission/permission.module.ts:26` |

`AuthorizationModule` is imported by `CommonModule` (`common.module.ts:8,33`), which is `AppModule.imports[1]` after `ConfigModule.forRoot` (`app.module.ts:57–62`). `PermissionModule` is imported by `PlatformModule` (`platform.module.ts:5,19`), which is `AppModule.imports[...]` far later (`app.module.ts:98`).

**2. NestJS 10.4.22 defines the ordering, and it is derivable without booting.**

`@nestjs/core/scanner.js`:

```js
async scan(module, options) {
    await this.registerCoreModule(options?.overrides);
    await this.scanForModules({ moduleDefinition: module, overrides: options?.overrides });   // (A)
    await this.scanModulesForDependencies();                                                   // (B)
    ...
}
async scanForModules({ moduleDefinition, ... }) {
    const { moduleRef: moduleInstance, ... } = (await this.insertOrOverrideModule(moduleDefinition, overrides, scope)) ?? {};  // insert FIRST
    ...
    for (const [index, innerModule] of modules.entries()) {                                     // then recurse into imports
        ...
        const moduleRefs = await this.scanForModules({ moduleDefinition: innerModule, ... });    // depth-first, imports order
        ...
    }
}
async scanModulesForDependencies(modules = this.container.getModules()) {
    for (const [token, { metatype }] of modules) {          // container Map => insertion order
        ...
        this.reflectProviders(metatype, token);
        ...
    }
}
reflectProviders(module, token) { ... providers.forEach(provider => { this.insertProvider(provider, token); this.reflectDynamicMetadata(provider, token); ... }) }
```

`reflectDynamicMetadata` dispatches to:

```js
// scanner.js:376
[constants_2.APP_GUARD]: (guard) => this.applicationConfig.addGlobalGuard(guard),
// application-config.js:66
addGlobalGuard(guard) { this.globalGuards.push(guard); }
```

and at request time:

```js
// guards/guards-context-creator.js:57,67
const globalGuards = this.config.getGlobalGuards();
...
return globalGuards.concat(scopedGuards);      // global first, then @UseGuards
```

Chain: **module insertion order (depth-first, root first, `imports` array order) × within-module `providers` array order → `globalGuards` push order → execution order; controller/method `@UseGuards` run last.** Because `scanForModules` inserts the current module *before* recursing into its imports, the root `AppModule` is inserted first.

**3. Derived runtime order.**

```
1. JwtAuthGuard            (AppModule — inserted first, providers[0])
2. PropertyScopeGuard      (AppModule — providers[1])
3. MultiTenantGuard        (AppModule — providers[2])
4. RolesGuard              (AuthorizationModule, via CommonModule — AppModule.imports[1])
5. PermissionGuard (common)
6. PropertyAccessGuard
7. DepartmentAccessGuard
8. WarehouseAccessGuard
9. PermissionGuard (platform)   (PermissionModule, via PlatformModule — AppModule.imports much later)
--- then scoped: @UseGuards(AuthGuard('jwt')) at 6 sites ---
```

**4. Consequence for the question asked: `JwtAuthGuard` runs first.** No route is under-guarded *because of ordering*.

**5. The downstream guards' actual dependencies are satisfied — and are fail-closed, not fail-open.**

- `common/authorization/permission.guard.ts:17-18` — `request.user?.permissions ?? []`, `request.user?.role`. If JWT had not run, `userPermissions = []` and `userRole = undefined` → `:23` `throw new ForbiddenException`. Ordering regression would produce **403s, not bypasses**.
- `platform/permission/permission.guard.ts:34-37` — `if (!userId) { … throw new ForbiddenException('Authentication required') }`. Explicitly fail-closed.
- `common/authorization/roles.guard.ts:17` — `request.user?.role` (optional chaining).

**6. Double-evaluation is real, and the two permission systems are disjoint.**

| Aspect | `common/authorization` | `platform/permission` |
|---|---|---|
| Decorator | `@Permissions(...)` | `@Permission(...)` |
| Metadata key | `'permissions'` (`permissions.decorator.ts:3`) | `'platform_permissions'` (`permission.decorator.ts:3`) |
| Call sites | **47** | **173** |
| Decision | Static list membership against `request.user.permissions`, with `ADMIN`/`SUPER_ADMIN` bypass (`common/…/permission.guard.ts:20-21`) | Async DB lookup via `PermissionResolverService.hasAllPermissions(userId, …, propertyId)` (`platform/…/permission.guard.ts:39`) |
| Behaviour when decorator absent | `return true` (`:14`) | `return true` (`:22`) |

Both are registered globally, so **both execute on every request**, each reading its own key. They do not conflict (different keys), but they do double-evaluate, and each is **opt-in**: a handler carrying neither decorator is authenticated but **not authorisation-checked**. Other decorator counts: `@PropertyScope(` = 29, `@Roles(` = 2, `@Public(` = 3.

**7. The controller-level guards are a separate, narrower layer:** 6 `@UseGuards(AuthGuard('jwt'))` sites — `auth.controller.ts:37,44,51,61`, `corporate-board.controller.ts:7`, `reporting-analytics.controller.ts:7`. They run *after* the nine globals and re-run JWT validation.

### Result

**`RESOLVED`**

Order is: `JwtAuthGuard → PropertyScopeGuard → MultiTenantGuard → the five AuthorizationModule guards → the platform PermissionGuard → scoped @UseGuards`. **Authentication precedes all authorisation, so no route is under-guarded by ordering.** Double-evaluation does occur (both permission guards run on every request), and both are opt-in.

**Confidence:** `Medium-High`. The derivation chain is quoted in full from the exact installed library and follows from two `push` calls plus one `concat`; what is *not* present is a runtime capture of the enhancer array. That residual is recorded in §16.

### Impact on the original finding

- **ARCH-053 (`UNCERTAIN / EVIDENCE REQUIRED`) → resolved.** The ordering is established and is safe with respect to the specific risk Document 01 named ("silently under-guarded").
- **ARCH-016 ("nine unsequenced global guards / two permission vocabularies") → retained, and now precisely specified.** "Unsequenced" is superseded by "sequenced, and the sequence is safe"; "two permission vocabularies" is confirmed with counts (47 vs 173 call sites, two metadata keys, two decision mechanisms, both global, both fail-open when unused).
- Three side-observations surfaced by this investigation (ineffective `PUBLIC_PATHS` regexes; webhook route requiring JWT; unauthenticated fallback to client-supplied tenant/property headers) are recorded in §16 as new evidence. They are **not** registered as findings here.

---

## 8. E-04 — Do platform HTTP routes resolve as documented or doubled?

**Evidence ID:** E-04
**Original finding:** ARCH-048 (`IMPLEMENTATION BUG`, Medium)
**Question (Document 01 §17.4):** "Do the platform HTTP routes resolve as documented or doubled? Static analysis says `/api/v1/api/v1/...` (ARCH-048) — Confirms whether the entire platform surface is unreachable."
**Evidence requested:** Single HTTP request against a booted API.

### Investigation performed

1. Read `main.ts` for global-prefix configuration, exclusions, and versioning.
2. Enumerated all `@Controller` declarations with an `api/v1/` literal.
3. Read the installed NestJS 10.4.22 source that concatenates the global prefix with a controller path.
4. Searched the whole repository for any consumer that uses the doubled path.
5. Checked `next.config.js` rewrites to confirm what path a browser actually emits.

### Evidence found

**1. Prefix configuration — no exclusion, no versioning.**

`apps/api/src/main.ts:21`:

```js
app.setGlobalPrefix('api/v1');
```

Signature used: one argument only. There is no `{ exclude: [...] }` object and no `app.enableVersioning(...)` call anywhere in `main.ts`. (Confirmed by reading the full bootstrap block, `main.ts:14–53`.)

**2. Twenty-one of 87 controllers hard-code the prefix.**

`rg -n "@Controller\('api/v1" apps/api/src` → **21 hits, all under `apps/api/src/platform/`**:

`user.controller.ts:9`, `property.controller.ts:9`, `organization.controller.ts:9`, `user-preference.controller.ts:7`, `theme.controller.ts:7`, `department.controller.ts:9`, `company.controller.ts:9`, `settings.controller.ts:7`, `localization.controller.ts:7`, `feature-flag.controller.ts:8`, `configuration.controller.ts:8`, `audit.controller.ts:8`, `activity-feed.controller.ts:7`, `timeline.controller.ts:7`, `tag.controller.ts:7`, `global-search.controller.ts:7`, `comment.controller.ts:7`, `attachment.controller.ts:7`, `branding.controller.ts:7`, `role.controller.ts:10`, `permission.controller.ts:9`.

Total `@Controller(` declarations in `apps/api/src`: **87** (matches Document 01).

**3. The installed framework concatenates unconditionally — this is the requested proof.**

`node_modules\.pnpm\@nestjs+core@10.4.22_...\@nestjs\core\router\route-path-factory.js`:

```js
if (metadata.globalPrefix) {
    ...
    return (0, shared_utils_1.stripEndSlash(metadata.globalPrefix || '') + path;
}
```

`routes-resolver.js` passes the prefix through unmodified for every controller:

```js
resolve(applicationRef, globalPrefix) { … this.registerRouters(controllers, moduleName, globalPrefix, modulePath, applicationRef); }
registerRouters(routes, moduleName, globalPrefix, modulePath, applicationRef) { … globalPrefix, … }
```

There is no branch that skips an already-prefixed `path`, no `path.startsWith(globalPrefix)` guard, and no exclusion list (none is configured). Therefore:

```
setGlobalPrefix('api/v1')  +  @Controller('api/v1/platform/users')
   =>  /api/v1  +  /api/v1/platform/users  =  /api/v1/api/v1/platform/users
```

**4. No consumer anywhere uses the doubled path.**

`rg -n "api/v1/api/v1" .` (excluding `node_modules`, `.next`, `dist`) → **2 hits, both in Document 01 itself** (`01_CURRENT_ARCHITECTURE_AUDIT.md:340` and `:686`). Zero hits in `apps/web`, `apps/admin`, `apps/mobile`, `packages`, `docs` outside Document 01.

**5. The web client's path confirms the shape a request takes.**

`apps/web/lib/api/client.ts:24` — `const API_BASE = process.env.NEXT_PUBLIC_API_URL || '/api/v1';`
`apps/web/next.config.js:57-61` — `async rewrites() { … destination: \`${API_URL}/:path*\` }`

A browser calling `<API_BASE>/platform/users` emits `/api/v1/platform/users`, which the framework will only match if the controller path were bare — it is not.

### Result

**`RESOLVED`**

The platform routes **are doubled**: every one of the 21 platform controllers resolves at `/api/v1/api/v1/platform/...`, not at `/api/v1/platform/...`. The entire documented platform surface is unreachable at its advertised path. No consumer in the repository uses either the documented path (it 404s) or the doubled path (it is referenced nowhere), so the surface is in practice unreachable *and* unused.

**Confidence:** `High`. The concatenation rule is quoted from the exact installed library build and is unconditional; the only thing absent is a single live HTTP round-trip, recorded as residual uncertainty in §16.

### Impact on the original finding

- **ARCH-048 (`IMPLEMENTATION BUG`, Medium) → confirmed at the framework-behaviour level, upgraded from "static analysis says" to "framework source says".** The finding's impact line ("Any future consumer must hard-code the doubled path or 404") is now established rather than predicted.
- No change to severity or to the "Safe to correct in place" classification recorded in Document 01 §14.1.
- One side-observation (the same prefix breaks `JwtAuthGuard.PUBLIC_PATHS`) is recorded in §16.

---

## 9. E-05 — Actual live database schema vs `schema.prisma`

**Evidence ID:** E-05
**Original finding:** ARCH-010 (`ARCHITECTURAL PROBLEM`, High), interacting with ARCH-009
**Question (Document 01 §17.5):** "What is the actual live database schema vs `schema.prisma`? 8,793 uncommitted lines (ARCH-010) — Determines the true migration baseline before any DB work."
**Evidence requested:** `prisma migrate diff` / `db pull` into a scratch file and diff.

### Investigation performed

1. Parsed all three Prisma schemas into their concrete table targets (`model` → `@@map` → `@@schema`).
2. Dumped every live base table across all three schemas.
3. Compared the two sets in both directions.
4. Compared the live `_prisma_migrations` ledger against the migration directories on disk and against what git tracks.
5. Checked, table by table, that the 15 live-only tables are genuinely unmodelled.
6. Traced two live tables that code depends on but migrations do not create.
7. Spot-checked column-level agreement on a small number of high-traffic tables.

### Evidence found

**1. Model→table comparison: the Prisma schemas are a strict subset of the live database.**

| Source | File | Models | → table targets |
|---|---|---|---|
| Main | `packages/db/schema.prisma` | 907 | 907 |
| Inventory | `packages/db/prisma/inventory.prisma` | 72 | 72 (`@@schema("xylo_inventory")`) |
| Platform | `packages/db/prisma/platform.prisma` | 39 | 39 (`@@schema("xylo_platform")`) |
| **Total** | | **1,018** | **1,018 unique** (no duplicate table mappings) |

| Live schema | Base tables |
|---|---|
| `public` | 922 |
| `xylo_inventory` | 72 |
| `xylo_platform` | 39 |
| **Total** | **1,033** |

```
=== IN PRISMA, ABSENT FROM LIVE DB (0) ===
=== IN LIVE DB, ABSENT FROM PRISMA (15) ===
_prisma_migrations          (Prisma internal)
allotment_pickups           availability_matrix_logs
banquet_event_sub_events    banquet_event_templates
banquet_event_waitlist      banquet_template_events
banquet_template_resources
currencies                  folio_postings
folios                      fx_rates
property_currency_settings  routing_instructions
trx_code_config
```

**Zero** Prisma models point at a missing table. **Fourteen** non-Prisma tables have no model.

Note on direction: Document 01's §17 framed this as "8,793 uncommitted lines". The *table-level* baseline is cleaner than that framing suggests — every modelled table exists. The drift is in the opposite direction: fourteen live tables are invisible to Prisma, so `prisma migrate diff --from-migrations --to-schema-datamodel` would propose **dropping** `folios`, `folio_postings`, `currencies`, `fx_rates`, `routing_instructions`, `trx_code_config`, `property_currency_settings`, `allotment_pickups`, `availability_matrix_logs` and six `banquet_*` tables.

**2. Migration ledger vs disk: exact 1:1 match — but git tracks fewer than half.**

| Measure | Value |
|---|---|
| Migration directories on disk (`Get-ChildItem packages/db/migrations -Directory`) | **51** |
| Distinct `migration_name` in live `_prisma_migrations` | **51** |
| Applied-but-no-directory | **0** |
| Directory-but-never-applied | **0** |
| Total rows in `_prisma_migrations` | 53 (see below) |
| Migration files tracked by git (`git ls-files packages/db/migrations`) | **24** — 23 × `migration.sql` + `migration_lock.toml` |
| Untracked migration directories (`git status --porcelain`) | **28** |
| `.gitignore` rules mentioning `migrations` | **none** — these are untracked by omission, not ignored |

The 53-vs-51 row difference is **two migrations recorded twice, one of each pair unfinished**:

```
…|UNFINISHED|20260817000000_add_checkin_sessions_and_guest_documents
…|2026-08-16 22:45:35.75228+00|20260817000000_add_checkin_sessions_and_guest_documents
…|UNFINISHED|20260910_b1_trx_codes
…|2026-09-29 11:20:17.19037+00|20260910_b1_trx_codes
```

Two rows have `finished_at IS NULL` with checksums identical to their finished counterparts — stale in-progress rows. `prisma migrate status` treats `finished_at IS NULL` as a failed migration requiring `prisma migrate resolve`.

**3. What git does *not* contain — the entire recent schema history.**

Tracked `migration.sql` files stop at `20260706000000_add_enterprise_outbox_messages`. Every directory from `20260714183730_remove_uuid_from_user_refs` onward is untracked, including the whole business-critical recent series:

```
20260910_b1_trx_codes                       20260911_b2_folios
20260912_b3_routing                         20260913_b4_fx
20260920_groups_blocks_allotments           20260927000000_availability_assertion_engine
20260928000000_availability_phase3_reservation_foundation
20260930000000_gba_baseline                 20261001000000_gba_pickup_canonicalization
20260726000000_phase0b_baseline             (+ 17 more)
```

Consequence (fact, not recommendation): a fresh clone of this repository contains **23 of the 51 migrations the live database has applied**, so the live schema cannot be reproduced from version control.

**4. The live database has already diverged from its own migration ledger.**

- `20260607002000_h26_email_outbox` **is** recorded as applied in `_prisma_migrations` and **does** `CREATE TABLE IF NOT EXISTS email_outbox` (`migration.sql:2`).
- Live: `to_regclass('public.email_outbox')` → **ABSENT**.
- `rg -i "DROP TABLE.*email_outbox" packages/db/migrations` → **0 hits**. No migration drops it.
- The drop therefore happened **outside** migration history. The application's own comment records where: `apps/api/src/infrastructure/email/email-outbox.worker.ts:13` — *"The HK migration (`hk_migration.sql:4157`) explicitly drops email_outbox."* — a hand-run script that is not in the migrations directory.
- Likewise `person_discrepancies` is referenced by **15 raw-SQL statements** in `apps/api/src/modules/housekeeping/infrastructure/repositories/person-discrepancy.repository.ts` (`:36,50,68,76,86,93,103,117,140,152` …) but the table is **ABSENT** live and **no migration in any of the 51 directories creates it** (`rg -i "person_discrepancies" packages/db/migrations` → 0 hits).

**5. One migration creates its table through a filename Prisma will never run.**

`packages/db/migrations/20260910_b1_trx_codes/companion.sql:2` — `CREATE TABLE trx_code_config (`. Prisma executes only `<name>/migration.sql`. The table exists live (it is in the 15-table list), so it was created outside Prisma's runner.

**6. Column-level spot checks (not exhaustive — see §16).**

`information_schema.columns` for `public.availability`:
`id:text, hotel_id:text, date:date, room_type:text, room_type_id:text, alloted:integer, reserved:integer, available:integer, oversell:integer, waitlist:integer, no_of_rooms:integer, inserted_at:timestamptz, updated_at:timestamptz` — 13 columns, consistent with `model availability` in `schema.prisma` (table exists per §1).

`xylo_inventory.*.property_id` across all 17 columns: **`uuid`** (see E-06, §10).

**7. `EmailOutboxWorker` is registered and default-enabled.**

`apps/api/src/modules/shared/shared.module.ts` providers include `EmailOutboxWorker`; `email-outbox.worker.ts:30-37` starts a 2-second-then-10-second poll in `onModuleInit` **unless** `EMAIL_OUTBOX_DISABLED === 'true'`.
- `.env:38` sets `EMAIL_OUTBOX_DISABLED=true` → disabled on this developer machine.
- `infra/k8s/base/deployments/api-deployment.yml` sets only `NODE_ENV`, `DATABASE_URL`, `REDIS_HOST`, `JWT_SECRET` (lines 35–47) → **does not set it**; `docker/api.Dockerfile` sets no such `ENV`.
- `apps/api/src/infrastructure/email/email-outbox.worker.ts:52-55` polls `FROM email_outbox`, which does not exist.

### Result

**`RESOLVED`** at the **table-set and migration-ledger level**, which is the level the question asks about ("the true migration baseline before any DB work").

- Prisma → live: **0 missing tables.**
- Live → Prisma: **14 unmodelled tables** (+ `_prisma_migrations`).
- Disk → ledger: **51/51 exact match**, but with **2 stale unfinished rows**.
- Git → disk: **23 of 51 migrations tracked**; 28 untracked, including every migration from 2026-07-14 onward.
- Ledger → live: **2 known divergences** (`email_outbox` dropped outside history; `person_discrepancies` never created).

Column-level comparison across all 1,018 models was **not** performed — `prisma db pull` rewrites `schema.prisma`, which is forbidden in this step. That residual is recorded in §16.

**Confidence:** `High` for everything in this section (set arithmetic over complete inventories, plus quoted file/line evidence). The column-level question is `UNABLE TO VERIFY` in full and is carried forward.

### Impact on the original finding

- **ARCH-010 ("Migration state unrecoverable from git", `ARCHITECTURAL PROBLEM`, High) → confirmed and quantified.** Every clause of the original finding now has a number attached: 51 dirs / **23 tracked migration files**, 28 untracked, 14 unmodelled live tables, 2 unfinished ledger rows, 1 table dropped outside history, 1 table referenced by 15 raw-SQL sites and never created. The finding's impact line ("No trustworthy baseline for any future schema change; drift is normalised") is established rather than asserted.
- **ARCH-009 (three Prisma schemas over one physical database) → unchanged.** The three-way split is confirmed intact (907/72/39 models over 922/72/39 live tables); no new evidence bears on ownership.
- **ARCH-050 (`EmailOutboxWorker`) → confirmed, with one fact corrected.** Document 01 recorded the trigger as "absent from `.env.example`"; the stronger and more operationally relevant fact is that `EMAIL_OUTBOX_DISABLED` **is absent from the Kubernetes deployment manifest**, so the worker is enabled on any deployment built from `infra/k8s`.
- No new finding is registered for `person_discrepancies` or for the `companion.sql` filename; both are recorded in §16.

---

## 10. E-06 — Is `xylo_inventory.property_id` Uuid or VarChar in the live DB?

**Evidence ID:** E-06
**Original finding:** ARCH-046
**Question (Document 01 §17.6):** "Is `xylo_inventory.property_id` actually Uuid or VarChar in the live DB? (ARCH-046) — Determines whether the stale client is causing live runtime errors today."
**Evidence requested:** `\d xylo_inventory."InvItem"`.

### Investigation performed

1. Queried `information_schema.columns` for every `property_id` / `hotel_id` column in `xylo_inventory`.
2. Read `propertyId` declarations in the current source schema `packages/db/prisma/inventory.prisma`.
3. Read `propertyId` declarations in the **installed** client's embedded schema `packages/db/node_modules/@prisma/inventory-client/schema.prisma`.
4. Checked presence in the live database of the four models Document 01 identified as deleted from source.
5. Counted application import sites of `@prisma/inventory-client`.

### Evidence found

**1. Live DB: `uuid`, without exception.**

```
select table_name||'.'||column_name||' = '||data_type
  from information_schema.columns
 where table_schema='xylo_inventory' and column_name in ('property_id','hotel_id');
```

All **17** matches, one per table, are `property_id = uuid`:

`inv_approval_workflow_rule`, `inv_audit_log`, `inv_brand`, `inv_contract`, `inv_department_store`, `inv_document_numbering_sequence`, `inv_integration_mapping`, `inv_inventory_ledger`, `inv_inventory_period`, `inv_item`, `inv_item_category`, `inv_module_configuration`, `inv_notification_rule`, `inv_purchase_order`, `inv_user_role_permission`, `inv_vendor`, `inv_warehouse`.

There are **zero** `hotel_id` columns in `xylo_inventory`.

**2. Current source schema agrees with the live DB.**

`packages/db/prisma/inventory.prisma`:

```
:21   propertyId  String?   @map("property_id") @db.Uuid
:41   propertyId String?            @map("property_id") @db.Uuid
:108  propertyId             String?   @map("property_id") @db.Uuid
```

**3. The installed client's embedded schema disagrees with both.**

`packages/db/node_modules/@prisma/inventory-client/schema.prisma`:

```
:21   propertyId  String?  @map("property_id") @db.VarChar(20)
:41   propertyId  String?  @map("property_id") @db.VarChar(20)
```

This is a `file:`-linked package (per Document 01 §9), so the generated client was built from `@db.VarChar(20)` while every live column is `uuid`.

**4. Four models in the installed client have no table live.**

```
to_regclass checks against xylo_inventory:
InvPurchaseRequestStatusHistory        -> ABSENT
InvPurchaseRequestApproval             -> ABSENT
InvInventoryReservationAllocation      -> ABSENT
inv_outbox_message                     -> ABSENT
```

All four are absent from `packages/db/prisma/inventory.prisma` (72 models ↔ 72 live tables, exact match per §9) but are present in the installed client — i.e. the installed client is **behind the source schema by four deleted models**.

**5. The stale client is in active production use.**

`rg -l "@prisma/inventory-client" apps/api/src` → **20+ files**, including `warehouse.controller.ts:5`, `goods-receiving-query.service.ts:4,7,8`, `goods-receiving.repository.ts:2`, `warehouse.repository.ts:2,8`, `warehouse-application.service.ts:3,4`, `goods-issue-application.service.ts:3`, `supply-request-approval.service.ts:2`, and others. It is not an unused artifact.

**6. "Causing live runtime errors today" — could not be tested.**

The API was not running during this step, no application log artifact exists in the repository, and no error output from this client is retained anywhere. Whether the `VarChar(20)`-vs-`uuid` mismatch and the four missing models produce runtime failures could not be observed.

### Result

**`RESOLVED`** for the type question:

**The live type is `uuid` on all 17 `xylo_inventory.property_id` columns. The current source schema (`@db.Uuid`) matches the live database. The installed, actively-imported client (`@db.VarChar(20)`) does not.**

**Sub-question — "is the stale client causing live runtime errors today": `UNABLE TO VERIFY`** (no API run, no retained logs). Carried to §16.

**Confidence:** `High` on the type question (single `information_schema` query plus two quoted schema lines).

### Impact on the original finding

- **ARCH-046 → confirmed at the type level.** The mismatch is now located precisely: *not* between source schema and database (those agree), but between the **installed generated client** and both. The finding is re-anchored on the installed client being stale by four models and wrong on one column type.
- Document 01 §14.1 classifies ARCH-046 as "Safe to correct in place". That classification is unaffected; the runtime-impact half remains unverified (§16).
- **ARCH-011 (two inventory pipelines, two clients) → reinforced factually**: the stale client is imported at 20+ sites, so the second pipeline is genuinely wired, not vestigial. No register change.

---

## 11. E-07 — Frontend consumers of platform endpoints

**Evidence ID:** E-07
**Original finding:** ARCH-028 ("unconsumed platform surface", classified in Document 01 §14.1 as "Not currently justified to change")
**Question (Document 01 §17.7):** "Frontend consumers of platform endpoints: static scan says none — Confirms ARCH-028 is truly unconsumed."
**Evidence requested:** Runtime network capture / access logs.

### Investigation performed

1. Searched every first-party client (`apps/web`, `apps/admin`, `apps/mobile`, `packages/**`) for all 21 platform path fragments.
2. Searched the entire repository for the doubled path produced by E-04.
3. Searched for any Next.js rewrite/proxy rule that would route to those paths.
4. Searched documentation outside Document 01 for client-facing references.
5. Attempted to locate any retained access-log or network-capture artifact.

### Evidence found

**1. Zero first-party references to any platform endpoint.**

`rg -n "platform/users|platform/roles|platform/properties|platform/organizations|platform/feature-flags|platform/settings|platform/permissions|platform/audit|platform/search|platform/tags|platform/comments|platform/attachments|platform/branding|platform/localization|platform/timeline|platform/activity-feed|platform/companies|platform/departments|platform/themes|platform/configuration"` across `apps/` and `packages/` → **21 hits, all of which are the `@Controller(...)` declarations themselves**, plus 3 unrelated internal-`ConfigurationService` imports (`shares-feature-flag.service.ts:2`, `walkin-defaults.service.ts:2`) and 1 test reading a source file (`t555-flag-mechanism.spec.ts:153`).

Searched separately and with identical scope:
- `apps/web` — **0**
- `apps/admin` — **0**
- `apps/mobile` — **0**
- `packages/**` — **0**

**2. Zero references to the doubled path** (E-04): 2 hits repo-wide, both inside Document 01.

**3. No rewrite rule routes there.** `apps/web/next.config.js:57-61` exposes a single catch-all rewrite `destination: \`${API_URL}/:path*\`` — a pass-through, not a platform-specific route. `apps/admin/next.config.js` has no rewrites.

**4. Documentation:** the only non-Document-01 file containing `api/v1/platform` is `docs/availability/phase-5/01_FORENSIC_AUDIT.md` — an internal forensic note, not a consumer.

**5. No runtime artifact exists.** There is no access log, HAR, or network capture anywhere in the repository (`rg -l "access.log|\.har\b|chrome-devtools"` → 0 relevant hits), and the API was not running during this step, so no live capture was possible.

**6. The route-shape argument makes the static result stronger than a static scan alone.** From E-04: a client calling the *documented* path `/api/v1/platform/...` gets a 404 (the route is `/api/v1/api/v1/platform/...`), and a client calling the *actual* path would have to hard-code the doubled form — which appears nowhere. There is therefore no path on which a working first-party consumer could exist without leaving a trace in the source.

### Result

**`RESOLVED`** for the question as posed about first-party consumers:

**No first-party frontend or package consumes any platform endpoint. ARCH-028 ("unconsumed platform surface") is confirmed.**

**Sub-question — external/third-party callers: `UNABLE TO VERIFY`.** No access log, no API run, no external traffic evidence was available. Carried to §16.

**Confidence:** `High` for first-party consumption (two independent negative searches plus the route-shape argument).

### Impact on the original finding

- **ARCH-028 → confirmed.** Document 01 §14.1's "Not currently justified to change — no functional harm today; revisit when those areas are activated" now rests on measured evidence rather than a static scan. The new fact from E-04 (the surface is also *unreachable* at its advertised path) is an impact on how ARCH-048 and ARCH-028 read together; both rows are left unchanged.
- No severity, classification, or register change.

---

## 12. E-08 — Do `apps/admin` / `apps/mobile` have production consumers?

**Evidence ID:** E-08
**Original finding:** ARCH-036 ("both appear to be prototypes")
**Question (Document 01 §17.8):** "Do `apps/admin`/`apps/mobile` have production consumers at all? Both appear to be prototypes (ARCH-036) — Determines whether they are in scope for correction or preservation."
**Evidence requested:** Deployment configuration / route usage evidence.

### Investigation performed

1. Enumerated every deployment manifest under `infra/k8s`.
2. Enumerated every image built by CI (`build-docker.yml` matrix and `docker/*.Dockerfile`).
3. Counted API call sites per app.
4. Located each app's API base configuration.
5. Checked for any other workflow that builds or deploys these apps.

### Evidence found

**1. Deployment manifests — three apps, no mobile.**

```
infra/k8s/base/deployments/
    admin-deployment.yml
    api-deployment.yml
    web-deployment.yml
    init-containers/prisma-migrate.yml
```

There is **no** `mobile-deployment.yml`, no `infra/k8s/**` reference to mobile, and no `expo`/`react-native` workload of any kind.

**2. CI builds exactly three images.**

`.github/workflows/build-docker.yml:16-17`:

```yaml
matrix:
  app: [api, web, admin]
```

`:32` — `file: docker/${{ matrix.app }}.Dockerfile`
`docker/` contains exactly `api.Dockerfile`, `web.Dockerfile`, `admin.Dockerfile`.

**3. API call sites per application.**

| App | `fetch(` / `axios` / `apiClient` / `useQuery(` call sites |
|---|---|
| `apps/web` | **278** |
| `apps/admin` | **1** |
| `apps/mobile` | **6** |

**4. API base configuration.**

- `apps/admin/app/(corporate)/inventory/page.tsx:8` — `const API_BASE = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:4000/api/v1';` → this is the **only** file in `apps/admin` containing `NEXT_PUBLIC_API_URL`.
- `apps/mobile/services/api/client.ts:4` — `const API_URL = process.env.EXPO_PUBLIC_API_URL || 'http://localhost:4000/api/v1';` with `baseURL: API_URL`
- `apps/mobile/hooks/useAuth.ts:7` — `fetch(\`${process.env.EXPO_PUBLIC_API_URL}/auth/login\`)`
- `apps/api/src/main.ts:46` — `const port = process.env.PORT || 4000;` (`.env:18` `PORT=4000`) → both defaults target the real API port.

**5. No other workflow builds either app.** Workflows present: `build-docker.yml`, `ci-test.yml`, `deploy-k8s.yml`, `infra-apply.yml`, `security-dast.yml`, `security-sast.yml`. The build matrix above is the only place app images are named.

### Result

**`RESOLVED`**

Both are prototypes, in different degrees:

- **`apps/admin` is deployed but almost entirely disconnected.** It has an image (`docker/admin.Dockerfile`), a manifest (`admin-deployment.yml`), and CI coverage — yet exactly **one** file in the entire app calls the API. Everything else in the corporate dashboard is local/static.
- **`apps/mobile` is connected but not deployed at all.** It has a working API client (6 call sites, correct default base URL) but **no Dockerfile, no image matrix entry, no Kubernetes manifest, and no build workflow**. There is no deployment path for it in this repository.

**Confidence:** `High` — deployment configuration is declarative and was read in full; call-site counts are exhaustive.

### Impact on the original finding

- **ARCH-036 → confirmed, and refined into two distinct sub-facts** rather than one blanket "both are prototypes":
  - admin = *deployed, unconsumed* (inverse of E-07's *consumed, undeployed* pattern in `apps/web`),
  - mobile = *consumed, undeployed*.
- This materially affects the "in scope for correction or preservation" question Document 01 posed: the two apps now have different answers on the evidence. Recorded as an impact statement only; ARCH-036's row and severity are unchanged.
- No new finding is registered; the mobile deployment gap is recorded in §16.

---

## 13. E-09 — BullMQ queue consumer liveness at runtime

**Evidence ID:** E-09
**Original finding:** interacts with §12 (Integration / Operational Architecture) and Document 01's queue wiring observations
**Question (Document 01 §17.9):** "BullMQ queue consumer liveness at runtime — 4 of 5 registered queues appear to have no consumer — Determines whether queued work is silently accumulating."
**Evidence requested:** Redis `LLEN` on each queue with the API running.

### Investigation performed

1. Enumerated every registered queue, producer, and consumer in source.
2. Checked which processor classes are actually registered in a NestJS module.
3. Ran a read-only Redis census over every queue: key existence, `LLEN wait/active`, `ZCARD completed/failed/delayed`.
4. Read the `removeOnComplete` configuration to determine whether an empty `completed` set is meaningful.
5. Checked whether the queues are currently backed up.

### Evidence found

**1. Registration, production, and consumption — full matrix.**

| Queue | Registered (`common/queue/queue.module.ts:8`) | Producer | Consumer class | Consumer **registered in a module**? |
|---|---|---|---|---|
| `events` | yes | `outbox-publisher.ts:18`, `event-publisher.ts:8` | `EventsConsumer` (`events.consumer.ts:23`, `@Processor('events')`) | **yes** — `shared.module.ts` providers |
| `inventory` | yes | **none** | **none** | — |
| `notifications` | yes | `notification.service.ts:13` | **none** | — |
| `sync` | yes | **none** | **none** | — |
| `audit` | yes | `audit-publisher.ts:8` | **none** | — |
| `analytics` | **no** | **none** | `AnalyticsConsumer` (`queue.consumers.ts:5`, `@Processor('analytics')`) | **no** — `rg "queue.consumers"` finds **0 module imports**; the only hit is a spec file |

Full producer inventory (`rg "InjectQueue('"`): `events` ×2, `notifications` ×1, `audit` ×1. That is all.
Full consumer inventory (`rg "@Processor\("`): exactly 2 classes in the entire codebase.
`shared.module.ts` providers: `XyloWebSocketGateway, OutboxProcessor, EventsConsumer, S3StorageService, EmailService, EmailOutboxWorker` — **`AnalyticsConsumer` is absent.**

**2. Redis census (read-only, API not running).**

```
events         meta=1 wait=0 active=0 completed=100 failed=0 delayed=0   keys=105
inventory      meta=1 wait=0 active=0 completed=0   failed=0 delayed=0   keys=1
notifications  meta=1 wait=0 active=0 completed=0   failed=0 delayed=0   keys=1
sync           meta=1 wait=0 active=0 completed=0   failed=0 delayed=0   keys=1
audit          meta=1 wait=0 active=0 completed=0   failed=0 delayed=0   keys=1
analytics      meta=1 wait=0 active=0 completed=0   failed=0 delayed=0   keys=1
```

Interpretation, with the configuration that makes it meaningful:

- `apps/api/src/common/queue/worker-base.ts:75` — `removeOnComplete: { count: 100 }`; `retry.strategy.ts:15` — `removeOnComplete: 100`. **Completed jobs are retained, capped at 100.** Therefore `completed = 100` for `events` means *"at cap"*, and `completed = 0` for the other five means **"no job has ever completed"** — not "jobs completed and were deleted".
- If a queue has **no worker**, jobs cannot leave `wait`. `wait = 0` on all six means **no queue is backed up**.
- `keys = 1` for the five non-`events` queues means the only key present is `meta` — the queue object was instantiated (which `BullModule.registerQueue` does for the five registered queues) but **no job was ever enqueued**.
- `events` at 105 keys and `completed = 100` proves the consumer is live and that jobs have flowed.

**3. Caveat on provenance of `bull:analytics:meta`.** `analytics` is not in `registerQueue` and its processor is not registered, yet a `meta` key exists. Its origin could not be attributed (possibly an earlier code revision or an ad-hoc `QueueFactory.getQueue`). Recorded in §16.

**4. Is anything accumulating?** **No.** `wait = 0` and `active = 0` on all six; `failed = 0` on all six. Nothing is silently piling up.

**5. `queue.factory.ts` / `worker-base.ts` are defined but never called.** `rg "createWorker\(|getQueue\("` finds only the definitions (`queue.factory.ts:21,29`) — no call sites outside the factory itself. The hand-rolled worker/DLQ path is dead; the live path is `@Processor`.

### Result

**`RESOLVED`**

Only **one** of six queues (`events`) has ever carried and completed jobs, and its consumer is live. Four queues (`inventory`, `sync`, `notifications`, `audit`) are registered, two of them have producers, and **none has a consumer**; one queue (`analytics`) has a consumer class that is **never registered as a provider**, so it never starts. **Nothing is accumulating** — the flip side being that `notifications` and `audit` have never received a single job despite having live producers.

**Confidence:** `Medium-High`. The wiring half (registration, producers, consumers, `removeOnComplete`) is `High` and fully static. The runtime half is `Medium-High`: the census is a persisted-state snapshot taken **with the API not running**, not a live capture as Document 01 requested. Residual in §16.

### Impact on the original finding

- The question behind Document 01's §17.9 ("is queued work silently accumulating?") is answered **no** — which is the operationally important half.
- The premise ("4 of 5 registered queues appear to have no consumer") is **confirmed and extended**: it is 4 of 5 registered queues *plus* a sixth unregistered queue whose consumer can never start.
- `notifications` and `audit` having producers but zero lifetime jobs is a new, quantified observation. Recorded in §16 as new evidence; no finding row is created here.
- Document 01's "BullMQ consumers have no dead-letter/retry visibility" observation is unaffected: `worker-base.ts:72` defines a DLQ queue that no live path reaches, and `failed = 0` across all queues means the retry/DLQ path has never been exercised.

---

## 14. E-10 — Is `AVAILABILITY_TEST_DATABASE_URL` ever set outside a developer machine?

**Evidence ID:** E-10
**Original finding:** Document 01 §11 (Testing Architecture); interacts with ARCH-024
**Question (Document 01 §17.10):** "Whether `AVAILABILITY_TEST_DATABASE_URL` is ever set outside a developer machine — Determines whether the 64 DB specs provide any CI safety."
**Evidence requested:** CI/CD environment inventory.

### Investigation performed

1. Read every file in `.github/workflows/` for DB service containers, env blocks, and the variable itself.
2. Enumerated every repository location that mentions the variable.
3. Checked `.env`, `.env.example`, `compose.yaml`, and all Kubernetes manifests.
4. Confirmed how `pnpm test` resolves under Turborepo.

### Evidence found

**1. Every workflow, in full.**

```
.github/workflows/
    build-docker.yml      ci-test.yml       deploy-k8s.yml
    infra-apply.yml       security-dast.yml security-sast.yml
```

`rg -n "AVAILABILITY_TEST" .github` → **0 hits.** None of the six workflows defines the variable.

**2. `ci-test.yml` has no database of any kind.**

```
name: CI - Test
jobs:                        (4 jobs, all `runs-on: ubuntu-latest`)
  lint         → pnpm install --frozen-lockfile; pnpm lint
  typecheck    → pnpm install --frozen-lockfile; pnpm typecheck
  test         → pnpm install --frozen-lockfile; pnpm test
  (prisma)     → pnpm install --frozen-lockfile; cd packages/db && npx prisma validate
```

- **No `services:` block** → no `postgres` service container is started.
- **No `env:` block** on any job → no `DATABASE_URL`, no `AVAILABILITY_TEST_DATABASE_URL`.
- `pnpm test` → `turbo run test` → `apps/api` + `apps/web` only (Document 01 §11).
- The prisma job runs `npx prisma validate` from `packages/db`, which validates the **default** schema only — it does not touch `prisma/inventory.prisma` or `prisma/platform.prisma`.

**3. Every repository location that mentions the variable.**

| Location | Nature |
|---|---|
| `.env.example:117-119` | Documentation + a literal default: `AVAILABILITY_TEST_DATABASE_URL="postgresql://xylo_user:password@localhost:5432/xylo_cloud?schema=public"` |
| `compose.yaml:29` | **Comment only**: `#   AVAILABILITY_TEST_DATABASE_URL   test/CI only — required for DB-gated Jest suites (see .env.example)` |
| `apps/api/src/.../availability-postgres.harness.ts:8,116` | Consumer — `if (!databaseUrl) throw new Error(...)` |
| 10 × `*.postgres.spec.ts` files | Consumers (reservations, group-allotment, availability) |
| `apps/api/src/common/config/__tests__/t555-flag-mechanism.spec.ts:197-199` | A spec that asserts the string appears in `compose.yaml` **and** `.env.example` — i.e. documentation is what is being tested, not an actual setting |
| `docs/**` | Prose describing manual developer recipes |

**Not present in:** `.env` (checked directly), any file under `.github/`, any `infra/k8s/**` manifest, any Dockerfile.

**4. Consequence.** The harness (`availability-postgres.harness.ts`) and all gated specs are driven solely by `describePostgres` / `if (!databaseUrl)` guards. With the variable unset, they skip. It is unset in CI by construction (no env block, no service), so every one of the DB-gated suites skips on every pull request.

**5. Corroboration from Document 01's own words**, retained unchanged: *"but `n` is not set in CI, so they all `describe.skip` on PRs."* (Document 01 §11; the garbled token is a rendering artifact of the variable name in that file.)

### Result

**`RESOLVED`**

**No. `AVAILABILITY_TEST_DATABASE_URL` is never set outside a developer machine.** It exists as a documented recipe in `.env.example` and a comment in `compose.yaml`, is asserted to exist as *text* by `t555-flag-mechanism.spec.ts`, and is consumed by the harness and 10+ specs — but it appears in **zero** workflow files, has no CI service container to point at, and is absent from `.env` and from every deployment manifest. The DB-gated suites therefore provide **no CI safety**; they run only when a developer exports the variable by hand.

**Confidence:** `High` — a negative result over a closed, enumerated set (6 workflows, all env files, all manifests).

### Impact on the original finding

- Document 01 §11's claim is **confirmed with exact mechanism**: the gap is not an oversight in one workflow, it is structural — there is no database service to connect to even if the variable were set.
- **ARCH-024 (test architecture) → reinforced factually.** A documented verification path (`t555-flag-mechanism.spec.ts:197`) actively *asserts the presence of the documentation string* rather than the setting, which is why the gap survives a green test suite. Recorded as an impact statement; no register row is changed.
- No new finding is registered.

---

## 15. Impact on Document 01 Findings Register

Document 01's register is **not** modified by this step. The table below records how the evidence changes how each affected row should be read.

| Finding | Document 01 status | Status after Step 02 | Nature of the change |
|---|---|---|---|
| **ARCH-051** | `UNCERTAIN / EVIDENCE REQUIRED` | **Evidence supplied — item closed** | RLS is enabled on 386 live tables with 392 policies; repository contains 1 policy file, 0 enabling statements, and that policy is absent live |
| **ARCH-052** | `UNCERTAIN / EVIDENCE REQUIRED` | **Evidence supplied — item closed** | Mutators identified: `trg_update_allotment_pickup` → `update_allotment_pickup()`, `trg_block_availability` → `block_availability_on_allotment()`; absent from every repo file |
| **ARCH-053** | `UNCERTAIN / EVIDENCE REQUIRED` | **Evidence supplied — item closed** | Order derived from NestJS 10.4.22 source; `JwtAuthGuard` runs first; no ordering-driven under-guarding |
| **ARCH-008** | `ARCHITECTURAL PROBLEM`, Critical | Retained; **characterisation sharpened** | Scale confirmed (386/392 vs 375) and enforcement posture newly established (inert: owner + `SUPERUSER` + `BYPASSRLS` + 0 `FORCE`) |
| **ARCH-007** | `ARCHITECTURAL PROBLEM`, Critical | Retained; **reinforced** | With RLS inert, header-based scoping is effectively the sole tenant boundary. Impact statement only |
| **ARCH-041** | listed under "Correctable with controlled migration" | Retained; **re-characterised** | DB-level writers exist but sit on tables the application no longer writes (`allotment_pickup` = 0 rows, `allotment` = 1); dual-writer coupling is live in DDL, dormant in traffic |
| **ARCH-016** | Technical debt / architectural | Retained; **specified** | "Unsequenced" → sequenced and safe; "two permission vocabularies" confirmed as two keys, 47 vs 173 sites, both global, both opt-in, both executing per request |
| **ARCH-048** | `IMPLEMENTATION BUG`, Medium | Retained; **upgraded from static analysis to framework proof** | `route-path-factory.js:39` concatenation quoted; no consumer uses either path |
| **ARCH-010** | `ARCHITECTURAL PROBLEM`, High | Retained; **quantified** | 51 dirs / **23 tracked** / 28 untracked / 14 unmodelled live tables / 2 unfinished ledger rows / 1 table dropped outside history / 1 table never created |
| **ARCH-046** | listed "Safe to correct in place" | Retained; **re-anchored** | Source schema and live DB **agree** (`uuid`); the installed `file:`-linked client disagrees (`VarChar(20)`) and is stale by 4 models; imported at 20+ sites. Runtime-error sub-question unverified |
| **ARCH-028** | "Not currently justified to change" | Retained; **confirmed** | Zero first-party consumers across `web`/`admin`/`mobile`/`packages`, plus no reachable path to consume |
| **ARCH-036** | `ARCHITECTURAL PROBLEM` (prototypes) | Retained; **split into two sub-facts** | admin: deployed (image + manifest + CI) but **1** API call site; mobile: **6** call sites but no image, no manifest, no workflow |
| **ARCH-050** | `IMPLEMENTATION BUG`, Medium | Retained; **one fact corrected** | The operative gap is that `EMAIL_OUTBOX_DISABLED` is absent from `infra/k8s/base/deployments/api-deployment.yml`, not merely from `.env.example`; target table confirmed ABSENT live |
| **ARCH-009** | `ARCHITECTURAL PROBLEM`, High | Unchanged | 907/72/39 models over 922/72/39 live tables — consistent with Document 01 |
| **ARCH-011** | `ARCHITECTURAL PROBLEM`, High | Retained; **reinforced** | Stale inventory client imported at 20+ sites — the second pipeline is wired, not vestigial |
| **ARCH-032** (Temporal) / §12 ops | — | Unchanged | Not the subject of any of the ten items |

**Register totals after Step 02:** unchanged at **53 findings** (6 SOUND, 26 ARCHITECTURAL PROBLEM, 8 TECHNICAL DEBT, 5 LEGACY RESIDUE, 5 IMPLEMENTATION BUG, 3 UNCERTAIN). The three `UNCERTAIN` rows (ARCH-051, ARCH-052, ARCH-053) now have their evidence satisfied in this document; their *classification* is a Document 03+ decision and is deliberately not changed here.

**Document 01 §14.1 "Evidence required first" bucket** (ARCH-051, ARCH-052, ARCH-053): the precondition for all three is now met.

---

## 16. Unresolved Items & New Evidence (Out-of-Scope for Step 02)

### 16.1 Sub-questions that remain `UNABLE TO VERIFY`

| Ref | Sub-question | Why it could not be closed |
|---|---|---|
| E-06 | Is the stale inventory client causing live runtime errors *today*? | API not running during this step; no application log artifact retained in the repository |
| E-07 | Are there external / non-repository callers of the platform surface? | No access log, HAR, or network capture exists in the repository; API not running |
| E-03 | Runtime confirmation of the enhancer array (as opposed to the source-derived order) | Booting the API would start pollers/publishers against the shared live database |
| E-04 | Single live HTTP round-trip confirming `/api/v1/api/v1/...` | Same — API not booting this step |
| E-05 | **Column-level** diff of all 1,018 models vs live | `prisma db pull` rewrites `schema.prisma` (forbidden in this step); `prisma migrate diff` writes a shadow DB |
| E-02 | Which actor updated `availability` on 2026-09-17 | API was not running; no audit trail exists on that table beyond `updated_at` |
| E-09 | Live (as opposed to persisted) queue census | API not running; census is last-run persisted state |

### 16.2 New evidence observed but outside the ten questions

Recorded for the record. **No finding row is created for any of these in this step.**

1. **DB-resident DDL with zero repository footprint.** Six trigger/function objects governing availability behaviour (`trg_block_availability`, `trg_update_allotment_pickup`, `update_allotment_pickup`, `block_availability_on_allotment`, `sp_refresh_availability`, `sp_daily_availability_refresh`) exist in the live database and appear in **no** file in the repository — not in migrations, not in `partitions/`, not in git history under those names.
2. **Two orphan functions with no possible caller.** `sp_daily_availability_refresh` and `sp_refresh_availability` have no caller in `pg_proc`, no trigger, and `pg_cron` is not installed — so no scheduled job exists in this database at all.
3. **`JwtAuthGuard.PUBLIC_PATHS` is dead code.** The six regexes (`jwt-auth.guard.ts:8-15`) are anchored as `^/auth/login`, `^/webhooks/...`, `^/health`, etc., but `request.path` includes the global prefix (`/api/v1/auth/login`), so **none can ever match**. The mechanism still works only because `@Public()` is applied at `auth.controller.ts:13,25` and `health.controller.ts:9`.
4. **Webhook ingress would require a JWT.** `webhook-ingress/webhook.controller.ts:4` declares `@Controller('webhooks')` with `@Post(':channel')` and carries **no** `@Public()` — while `JwtAuthGuard.PUBLIC_PATHS` contains `/^\/webhooks\/.+/`, which cannot match the prefixed path. On the evidence, `POST /api/v1/webhooks/:channel` falls through to JWT validation. (The retired `/rates/engine/quote` and `/rates/engine/book` entries in the same list point at routes that no longer exist.)
5. **Unauthenticated fallback to client-supplied identity headers.** `property-scope.guard.ts:41-43` — when `request.user` is absent, `propertyId` is taken from `x-property-id` or `:propertyId`; `multi-tenant.guard.ts:34-36` — when unauthenticated, `tenantId` is taken from `x-tenant-id`. Both guards also short-circuit on `@Public()` (`property-scope.guard.ts:17`, `multi-tenant.guard.ts:15`) and on `required === false` (`:23` / `:21` respectively).
6. **Two permission guards execute on every request.** Distinct classes, distinct metadata keys (`'permissions'` vs `'platform_permissions'`), distinct decision mechanisms (static list vs async DB resolution), both registered via `APP_GUARD`, both returning `true` when their decorator is absent.
7. **The one RLS policy the repository does declare does not exist live.** `packages/db/migrations/20260607003000_country_scoped_tax_codes/migration.sql:62` creates `p_tax_codes`; live `pg_policies` has **0** policies on `tax_codes` (which nevertheless has `relrowsecurity = true`).
8. **Two stale unfinished rows in `_prisma_migrations`.** `20260817000000_add_checkin_sessions_and_guest_documents` and `20260910_b1_trx_codes` each have a `finished_at IS NULL` row alongside a finished row with an identical checksum — the state `prisma migrate status` reports as failed and requires `migrate resolve`.
9. **A Prisma migration filename the runner will never execute.** `packages/db/migrations/20260910_b1_trx_codes/companion.sql` creates `trx_code_config`; Prisma executes only `migration.sql`.
10. **`person_discrepancies` is referenced 15 times in production code and has never existed.** No table live, no migration creating it, in any of the 51 directories.
11. **Mobile has no deployment path.** No Dockerfile, no CI matrix entry, no Kubernetes manifest — while `apps/admin`, with one API call site, does have all three.
12. **Queue wiring gaps:** `notifications` and `audit` have live producers and zero lifetime jobs and no consumer; `inventory` and `sync` are registered with neither; `analytics` has a consumer class with no module registration and no `registerQueue` entry (its lone Redis `meta` key is unattributed).
13. **`queue.factory.ts` / `worker-base.ts` are never called.** The hand-rolled worker + DLQ path (`worker-base.ts:72`) has no call sites; the live path is `@Processor` only.
14. **`bull:analytics:meta` provenance unexplained.** The queue is not registered in `queue.module.ts` yet has a Redis `meta` key.

### 16.3 Facts deliberately not acted upon

Per this step's constraints, none of the above was fixed, and no plan, sequence, owner, or target state was drafted for any of it. Fourteen of Document 01's 53 findings have revised characterisations (§15); their classification, severity, and repairability remain exactly as Document 01 recorded them.

---

*End of Document 02. No source code, database schema, migration, API, business logic, CI configuration, deployment manifest, or frontend behaviour was modified in the course of this evidence step. All database interaction was read-only `SELECT` against catalogue and data tables. No architecture documents other than this one were created.*

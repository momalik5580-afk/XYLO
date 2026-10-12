# XYLO Phase 3 — Tax Database Contract Gap Analysis & Schema Design

**Status:** ANALYSIS ONLY — no code, schema, or migration changes made.
**Schema scope:** `public` only. `xylo_platform` and all non-`public` schemas are OUT OF SCOPE.
**Live database:** NOT CONNECTED. All findings derive from repository files (Prisma models, migration SQL, application code, tests). Live-schema verification is marked NOT PERFORMED throughout.
**Date:** 2026-10-12

---

## Executive Summary

The tax domain has **three coexisting, disconnected calculation engines** and **one authoritative service (`TaxService`) that is blind to its own per-hotel configuration tables**. The schema already contains the structures needed for per-hotel tax configuration — `hotel_tax_config` (activation + effective dates) and `tax_rates` (date-scoped rate overrides) — but **no production code reads either table**. The Hotel Settings "Taxes" UI is a hardcoded mock with no HTTP. `folio_tax_details` is keyed to the folio, not the charge, so multiple charges overwrite each other's tax records.

The good news: the intended architecture is **already documented in migration `20260607003000_country_scoped_tax_codes`**, whose header comment states the design verbatim — country catalog (`tax_codes`) + per-hotel config (`hotel_tax_config`, `tax_rates`). The gap is application wiring, not missing schema, for most requirements.

---

# A. Existing Database Contract Inventory

## A.1 Tax configuration tables

### `tax_codes` — country-level tax catalog
- **Defined:** Prisma `schema.prisma:12280-12295`. Migration `20260607003000_country_scoped_tax_codes` (re-parented from hotel-scoped to country-scoped). Baseline `20250101000000_c03_core_baseline_capture`.
- **PK:** `(tax_code, country)`.
- **Columns:** `tax_code varchar(20)`, `country varchar(50) default ''`, `tax_description varchar(100)`, `tax_percentage decimal(5,2)`, `tax_type varchar(20) nullable`, `active_yn char(1) default 'Y'`, `created_at`, `updated_at`.
- **Scope:** Country.
- **Read by:** `TaxService.getActiveTaxCodes` (`apps/api/src/core/tax/tax.service.ts:114-117`) via `findMany({ where: { country, active_yn: 'Y' } })`.
- **Written by:** No production code.
- **Live verification:** NOT PERFORMED.

### `hotel_tax_config` — per-hotel activation + effective dates
- **Defined:** Prisma `schema.prisma:5261-5272`. Baseline migration `:7243-7261`. FK re-pointed in `20261002000000_c03_late_reconciliation:3045`.
- **PK:** `config_id` (serial, map `hotel_id_tax_config_pkey`).
- **Columns:** `hotel_id varchar(20)`, `tax_code varchar(20)`, `country varchar(50)`, `effective_from date NOT NULL`, `effective_to date nullable`, `active_yn char(1) default 'Y'`.
- **FK:** `(tax_code, country)` → `tax_codes(tax_code, country)` (map `resort_tax_config_tax_code_country_fkey`).
- **Index:** `idx_hotel_tax_config_tax_code` on `(tax_code)`.
- **RLS:** Enabled at baseline `:28041`. No `CREATE POLICY` found in repository migrations — access depends on the connecting role's privileges (owner/bypassrls). NOT PERFORMED: live policy verification.
- **Scope:** Hotel.
- **Read by:** **Nothing in `apps/api`.** Grep for `hotel_tax_config` / `hotelTaxConfig` in `apps/api/src` returns zero production hits.
- **Written by:** Nothing.
- **Migration intent (verbatim, `20260607003000:10-11`):** "The 4 hotel-level config tables (`hotel_tax_config`, `tax_rates`, `tax_reporting`, `tax_summary`) keep their `hotel_id` (audit trail / per-hotel configuration) but gain a `country` column so the FK to `tax_codes` can be re-pointed."
- **Classification:** **EXISTING BUT UNUSED.**

### `tax_rates` — per-hotel rate overrides with date ranges
- **Defined:** Prisma `schema.prisma:12314-12329`.
- **PK:** `tax_rate_id` (serial).
- **Columns:** `tax_code varchar(20)`, `hotel_id varchar(20)`, `country varchar(50)`, `begin_date date NOT NULL`, `end_date date nullable`, `tax_percentage decimal(5,2)`, `created_at`, `updated_at`.
- **Unique:** `(tax_code, hotel_id, country, begin_date)` and `(tax_code, hotel_id, begin_date)`.
- **FK:** `(tax_code, country)` → `tax_codes(tax_code, country)` (map `tax_rates_tax_code_country_fkey`), `onDelete: Cascade`.
- **Index:** `idx_tax_rates_dates` on `(begin_date, end_date)`.
- **Scope:** Hotel, date-scoped.
- **Read by:** **No DB read.** The only `tax_rates` type reference is `crs-engine.service.ts:186` — an optional caller-supplied `TaxRate[]` array for backward-compat, never loaded from this table.
- **Written by:** Nothing.
- **Classification:** **EXISTING BUT UNUSED.**

### `hotels` — hotel master (country resolution)
- **Defined:** Prisma `schema.prisma:5275+`.
- **Relevant columns:** `hotel_id varchar(20)` PK, `country varchar(50) nullable`.
- **Read by:** `TaxService.getActiveTaxCodes` via raw SQL `SELECT country FROM hotels WHERE hotel_id = $1` (`tax.service.ts:105-107`).
- **Note:** `country` is nullable. A hotel with NULL country yields **zero active tax codes** — a legitimate zero-tax outcome per the engine contract (`tax.service.ts:109-112`), not an error.

## A.2 Financial persistence tables

### `folio_tax_details` — persisted tax breakdowns (historical)
- **Defined:** Prisma `schema.prisma:4432-4445`.
- **PK:** `id` (text, UUID).
- **Columns:** `hotel_id`, `folio_id text`, `tax_type varchar`, `tax_rate decimal(5,2)`, `tax_amount decimal(15,2)`, `taxable_amount decimal(15,2)`, `created_at`, `updated_at`.
- **FK:** `hotel_id` → `hotels(hotel_id)` (map `fk_folio_tax_details_hotel_id`), `onDelete: Cascade`.
- **Index:** `(hotel_id, folio_id)`.
- **Critical structural fact:** **`folio_id` is text, and there is NO foreign key to `folio_charges` or `folio_postings`.** The table is **folio-scoped, not charge-scoped**.
- **Written by:** `TaxService.recordFolioTaxDetails` (`tax.service.ts:79-95`) via raw SQL: `DELETE FROM folio_tax_details WHERE hotel_id=$1 AND folio_id=$2` then re-`INSERT` one row per non-zero tax component.
- **Read by:** `BillingService.getFolioDetails` (`billing.service.ts:83-99`) for the `taxSummary`.
- **Classification:** **HISTORICAL DATA INTEGRITY GAP** — the DELETE-then-INSERT pattern is folio-wide, so a second charge on the same folio destroys the first charge's tax breakdown.

### `folio_charges` — charge lines
- **Defined:** Prisma `schema.prisma:4254-4285` (`@ignore`d relation to `folio_header`). Partitioned monthly (`folio_charges_2026_01` … `_default`).
- **PK:** `trx_no` (serial).
- **Columns:** `folio_id int`, `trx_date`, `trx_code varchar(20)`, `hotel_id varchar(20)`, `trx_description varchar(200)`, `trx_amount decimal(12,2)`, `trx_type varchar`, `posted_by`, `source_ref_id`, `source_ref_type`, commission/package fields.
- **`trx_type` values observed in code:** `'CHARGE'`, `'ADJUSTMENT'`, `'CREDIT'`.
- **Written by:** `BillingService.postCharge` (`billing.service.ts:185-189`, gross incl. tax), `BillingService.postCredit` (`:226-229`, net adjustment), activities/banquet `postFolioCharge`.
- **No tax columns** — tax is folded into `trx_amount` (gross) with the breakdown stored separately in `folio_tax_details`.

### `folio_postings` — posting ledger (cashiering / checkout auto-post)
- **Defined in migrations only** — baseline `:6076-6094`, `20260911_b2_folios`. **NOT in the Prisma schema** (`rg "model folio_postings"` returns nothing).
- **Columns:** `id text` PK (UUID default), `hotel_id varchar(20)`, `folio_id text`, `trx_code varchar(20)`, `trx_date date`, `amount numeric(12,2)`, `currency varchar(3) default 'USD'`, `base_amount numeric(12,2)`, `fx_rate numeric(12,6)`, `quantity numeric(12,3) default 1`, `description varchar(200)`, `guest_record_id`, `routing_rule_id`, `trx_no integer`, `posted_by text`, `reversed_by_posting_id text`, `created_at`.
- **Reversal support:** `reversed_by_posting_id` column exists; no production code observed writing it.
- **Written by:** check-out auto-post (`check-out.handler.ts:400-406`, `trx_code='TAX'` and `'ROOM'`), cashiering `post-charge.handler.ts:113-119`.
- **Classification:** **CONTRACT MISMATCH** — absent from the Prisma client; all access is raw SQL.

### `folio_header` — folio balances
- **Defined:** Prisma `schema.prisma:4354-4386`.
- **PK:** `folio_id` (int serial).
- **Columns:** `hotel_id`, `reservation_id`, `total_charge`, `total_credit`, `balance`, `status`, plus agent/net-rate fields.
- **Written by:** `BillingService` (`postCharge` `:193-196`, `postCredit` `:231-234`, `postPayment` `:151-154`).
- **Note:** `total_charge` stores **gross** (tax-inclusive) amounts for taxed charges. `balance` is derived, not recomputed from `folio_tax_details`.

### `tax_groups` — cashiering parallel tax config
- **Defined:** Prisma `schema.prisma:9822-9835`. `@@map("tax_groups")`.
- **PK:** `id` (uuid). **Unique:** `(hotelId, code)`.
- **Columns:** `hotelId varchar(20)`, `code varchar(20)`, `name varchar(100)`, **`components Json`** (`[{name, percent, seq}]`), `isActive`.
- **Scope:** Hotel.
- **Written by:** cashiering `create-tax-group.handler.ts:26-28`, `update-tax-group.handler.ts:25-32`.
- **Read by:** cashiering `post-charge.handler.ts:79-91` (via `transaction_codes` → `trx_code_config` → `tax_groups` join), `list-tax-groups.handler.ts:15-16`, `list-transaction-codes.handler.ts:25`.
- **Classification:** A **second, independent per-hotel tax configuration store**, keyed to transaction codes rather than to the country catalog. Its `components` JSON always computes **cumulative** (each component bases on the running total — `posting-domain.service.ts:46-54`), unlike `TaxService`'s SUBTOTAL/CUMULATIVE mix.

### `transaction_codes` / `trx_code_config` — cashiering code config
- **`transaction_codes`:** Prisma `schema.prisma:12574`.
- **`trx_code_config`:** **NOT in the Prisma schema.** Referenced only by raw SQL in `post-charge.handler.ts:82` and `list-transaction-codes.handler.ts:25`. **CONTRACT MISMATCH.**

### `room_types` — third tax engine input
- **Defined:** Prisma `schema.prisma:10953+`. **Has NO `tax_rate` column** in the Prisma model.
- **Referenced by:** `reconcile-room-tax.handler.ts:38,59` raw-SQLs `rt.tax_rate` via `LEFT JOIN room_types rt`.
- **Classification:** **CONTRACT MISMATCH** — the raw SQL references a column absent from the Prisma model. If the live table lacks the column, this handler fails at runtime. NOT PERFORMED: live column verification. Note `check-out.handler.spec.ts:498-503` asserts check-out no longer uses `room_types.tax_rate`, confirming the project already treats this path as retired — yet the group-allotment reconcile handler still queries it.

### `reservations` — used by third engine
- `room_rate`, `arrival_date`, `departure_date`, `group_block_id`, `hotel_id` read by `reconcile-room-tax.handler.ts:35-46`.

## A.3 Tax calculation engines (three, disconnected)

| Engine | Config source | Semantics | Consumers |
|---|---|---|---|
| **`TaxService`** (`core/tax/tax.service.ts`) | `hotels.country` → `tax_codes` (country catalog only) | SUBTOTAL vs CUMULATIVE via `classifyTaxType`; FLAT vs PERCENTAGE; round2 half-up | billing `postCharge`/`postCredit`, check-out auto-post, activities/banquet, CRS quote, `GET front-office/tax-config` |
| **`PostingDomainService`** (`cashiering/domain/posting-domain.service.ts:33-60`) | `tax_groups.components` JSON via `trx_code_config` | Always cumulative; percent-only; seq-ordered | cashiering `post-charge` |
| **`room_types.tax_rate`** (group-allotment) | `room_types.tax_rate` column (absent from Prisma) | `roomCharge * taxRate` (fraction, not percent) | `reconcile-room-tax.handler.ts:59-61` |

## A.4 Frontend contract

- **UI:** `apps/web/app/(dashboard)/settings/page.tsx`.
  - Tab registered `:42` (`{ id: 'taxes', label: 'Taxes' }`).
  - Hardcoded data `:88-93` — 4 literal objects (`VAT-10`, `TOUR-5`, `SVC-8`, `CITY-FIX`); `City Tax: '$5/night'` is a display string, not a numeric rate.
  - Static table render `:563-588`; Edit button `:582` has **no `onClick`**.
  - `saveHotelConfig` `:189-196` — in-memory Zustand store + `setTimeout(600)`. **Zero HTTP.**
  - Only real HTTP in the file is PBX (`api.get/put('/pbx-config/...')` `:161,172`).
- **Classification:** **UI/API INTEGRATION GAP.**

## A.5 API contract

- **`GET front-office/tax-config`** (`front-office.controller.ts:146-149`) → `CrsFrontDeskIntegrationService.getTaxConfig` (`crs-front-desk-integration.service.ts:31-44`).
  - Returns `{ taxCode, description, percentage, type: 'FLAT'|'PERCENTAGE' }[]`.
  - Reads `TaxService.getActiveTaxCodes(hotelId)`; hotel from `requireHotelId()` (ALS).
  - **No `@Permissions`, no `@Permission`, no `@Roles`, no `@Public`.** Controller is `@PropertyScope(false)` (`:21`).
  - **No write endpoint exists.**
- **Cashiering tax-groups CRUD:** `GET/POST/PUT` under cashiering controller (`cashiering.controller.ts:50`) — operates on `tax_groups`, the parallel store.

---

# B. Application-to-Database Gap Register

| ID | Required behavior | Existing contract | Expected contract | Classification | Evidence | Severity | Correction | Schema change needed? |
|----|-------------------|-------------------|-------------------|----------------|---------|----------|------------|----------------------|
| **TX-01** | Hotel admins load/save per-hotel tax settings | `hotel_tax_config` + `tax_rates` exist, unused | API reads/writes both, scoped by hotel | **EXISTING BUT UNUSED** | `schema.prisma:5261,12314`; zero `apps/api` refs | HIGH | Wire `TaxService` + add settings endpoints | **No** |
| **TX-02** | Calculation engine honors per-hotel activation | `getActiveTaxCodes` queries `tax_codes` by country only (`tax.service.ts:114-117`) | Joins `hotel_tax_config` for activation + effective dates | **APPLICATION LOGIC GAP** | `tax.service.ts:97-118` | HIGH | Extend `getActiveTaxCodes` with overlay | **No** |
| **TX-03** | Calculation engine honors per-hotel rate overrides | `tax_rates` unused | Date-aware rate override applied over catalog % | **APPLICATION LOGIC GAP** | `tax_rates` no DB readers | HIGH | Apply `tax_rates` in `getActiveTaxCodes` | **No** |
| **TX-04** | UI persists tax settings via API | Mock const + in-memory save | React Query load/save against new endpoints | **UI/API INTEGRATION GAP** | `settings/page.tsx:88-93,189-196,563-588` | HIGH | Replace mock with API calls | **No** |
| **TX-05** | Per-charge tax breakdowns | `folio_tax_details` keyed `(hotel_id, folio_id)`, DELETE-then-INSERT folio-wide (`tax.service.ts:86`) | Keyed per charge/posting | **HISTORICAL DATA INTEGRITY GAP** | `schema.prisma:4432`; `tax.service.ts:86-94`; `billing.service.ts:191,246` | **CRITICAL** | Add `trx_no`/`posting_id` FK column to `folio_tax_details` | **Yes** |
| **TX-06** | Second charge does not destroy first charge's tax detail | Same as TX-05 | Independent rows per charge | **HISTORICAL DATA INTEGRITY GAP** | `billing.service.ts:191` then `:246` both DELETE folio-wide | **CRITICAL** | Same as TX-05 | **Yes** |
| **TX-07** | Tax applies to gross-inclusive posting with auditable net | `folio_charges.trx_amount` = gross; net only recoverable by subtracting `folio_tax_details` | Explicit net + tax columns, or reliable charge linkage | **MISSING COLUMN OR CONSTRAINT** (partially) | `billing.service.ts:166,185-189` | MEDIUM | Solved by TX-05 linkage | Yes (via TX-05) |
| **TX-08** | `folio_postings` reachable via Prisma | Table in migrations, absent from Prisma schema | In Prisma client | **CONTRACT MISMATCH** | `rg "model folio_postings"` → none; baseline `:6076` | MEDIUM | Add Prisma model | **Yes** (model-only, no DDL) |
| **TX-09** | `trx_code_config` reachable via Prisma | Absent from Prisma schema; raw SQL only | In Prisma client | **CONTRACT MISMATCH** | `post-charge.handler.ts:82` | LOW | Add Prisma model | **Yes** (model-only) |
| **TX-10** | Group reconcile uses authoritative tax engine | `reconcile-room-tax` uses `room_types.tax_rate` (absent from Prisma) | Use `TaxService.computeTax` | **CONTRACT MISMATCH** + **APPLICATION LOGIC GAP** | `reconcile-room-tax.handler.ts:38,59-61`; `room_types` has no `tax_rate` | HIGH | Repoint handler to `TaxService` | **No** |
| **TX-11** | Cashiering tax config and `TaxService` config agree | Two independent stores (`tax_groups` vs `tax_codes`+overlays) with different semantics (always-cumulative vs SUBTOTAL/CUMULATIVE) | One authoritative config | **CONTRACT MISMATCH** | `posting-domain.service.ts:46-54` vs `tax.service.ts:51-61` | HIGH | **Unresolved business decision** — see C.6 | Possibly |
| **TX-12** | Config changes take effect immediately | `invalidateCache` exists (`tax.service.ts:122-125`) but **has zero production callers** | Called after every config write | **APPLICATION LOGIC GAP** | grep: no callers | MEDIUM | Call on settings save | **No** |
| **TX-13** | Fixed-amount taxes representable | `tax_percentage decimal(5,2)` reused as flat amount when `tax_type` in (`FLAT`,`FIXED`); `tax.types.ts` declares `FLAT`; UI mock has `$5/night` | Explicit amount + period unit | **MISSING COLUMN OR CONSTRAINT** (weak) | `tax.service.ts:36,56-58`; `tax.types.ts:1` | LOW | Acceptable as-is; note limitation | No (documented limitation) |
| **TX-14** | Effective-date-aware activation | `hotel_tax_config.effective_from/_to` exist, unused | Engine filters by business date | **EXISTING BUT UNUSED** | `schema.prisma:5266-5267` | HIGH | Part of TX-02 | **No** |
| **TX-15** | Currency consistency | `TaxBreakdown.currency` defaults `'USD'` (`tax.service.ts:6`); `folio_postings.currency` default `'USD'`; `folio_tax_details` **has no currency column** | Currency on tax detail rows | **MISSING COLUMN OR CONSTRAINT** | `schema.prisma:4432-4445` | LOW | Add `currency` with TX-05 | Yes (with TX-05) |
| **TX-16** | Rounding documented and consistent | `round2` half-up (`tax.service.ts:151-153`); cashiering uses `Math.round(x*100)/100` — equivalent | Single rounding policy | **CONTRACT MISMATCH** (benign) | both engines | LOW | Document; no change | No |
| **TX-17** | Authorization on tax endpoints | `GET front-office/tax-config` has no permission decorator; controller `@PropertyScope(false)` | Enforced read/write permissions | **APPLICATION LOGIC GAP** + **BLOCKED** (see §7) | `front-office.controller.ts:21,146` | HIGH | Out of DB-contract scope | No |

---

# C. Proposed Target Schema Design

## C.1 Structures to reuse as-is (no change)

1. **`tax_codes`** — remains the country-level catalog. Sufficient for code, description, percentage, type, active flag.
2. **`hotel_tax_config`** — remains the per-hotel activation + effective-date overlay. Its columns (`hotel_id`, `tax_code`, `country`, `effective_from`, `effective_to`, `active_yn`) fully express "which country taxes apply to this hotel, from when until when."
3. **`tax_rates`** — remains the per-hotel, date-scoped rate override. Its unique `(tax_code, hotel_id, country, begin_date)` correctly prevents duplicate overrides for the same start date and its `end_date` allows ranges.
4. **`hotels.country`** — the country-resolution key, unchanged.
5. **`folio_header`, `folio_charges`, `folio_postings`** — financial ledger, unchanged in structure.

**Conclusion on existing tables:** For requirements TX-01/02/03/12/14 (per-hotel configuration and calculation), the existing schema is **sufficient**. No new configuration table is warranted.

## C.2 Existing structures requiring change

### C.2.1 `folio_tax_details` — add charge/posting linkage (TX-05, TX-06, TX-07, TX-15)

**Business purpose:** Persist a tax breakdown that belongs to one specific charge or posting, so multiple charges on a folio retain independent, immutable tax records, and so a credit/refund can reference the original taxed posting.

**Why existing schema cannot meet this:** `folio_tax_details` has only `(hotel_id, folio_id)`. `recordFolioTaxDetails` DELETEs all rows for the folio then re-INSERTs (`tax.service.ts:86`). Posting charge A (tax breakdown A) then charge B (tax breakdown B) leaves only B's rows. The tax on A becomes unattributable. Additionally there is no FK to any charge table, so `taxable_amount` cannot be reconciled against a specific `trx_amount`.

**Proposed columns (additive, nullable for backfill safety):**

| Column | Type | Constraint | Purpose |
|---|---|---|---|
| `charge_trx_no` | `integer` nullable | FK → `folio_charges(trx_no)` `ON DELETE SET NULL` | Links detail to the `folio_charges` row (billing path) |
| `posting_id` | `text` nullable | FK → `folio_postings(id)` `ON DELETE SET NULL` | Links detail to the `folio_postings` row (checkout auto-post / cashiering path) |
| `currency` | `varchar(3)` nullable default `'USD'` | — | Currency of `tax_amount`/`taxable_amount` (TX-15) |
| `tax_code` | `varchar(20)` nullable | — | Distinct from `tax_type`; `tax_type` currently stores the tax *code* (`tax.service.ts:92` writes `tax.taxCode` into `tax_type`), which is a semantic overload |

**Integrity rules:**
- Exactly one of `charge_trx_no` / `posting_id` should be set for new rows (CHECK constraint, deferred).
- Retain the existing `(hotel_id, folio_id)` index for folio-summary reads; add index on `(hotel_id, charge_trx_no)` and `(hotel_id, posting_id)`.
- Remove the folio-wide `DELETE` in `recordFolioTaxDetails`; replace with per-charge delete (`WHERE hotel_id=$1 AND charge_trx_no=$2`) or upsert.

**Backfill implication:** Existing rows have neither linkage column and remain valid as folio-level historical snapshots. Nullable columns avoid a blocking backfill.

**Application behavior depending on it:** `BillingService.postCharge` (`billing.service.ts:191`), `BillingService.postCredit` (`:246`), `BillingService.getFolioDetails` (`:83-99`), `TaxService.recordFolioTaxDetails` (`:79-95`).

### C.2.2 `tax_codes.tax_percentage` semantics (TX-13) — documented limitation, no change

`tax_percentage decimal(5,2)` is overloaded: percent for `PERCENTAGE`, absolute amount for `FLAT`/`FIXED`. This is workable but should be documented in the column comment. A separate `flat_amount` column is **not** proposed — it would duplicate meaning and complicate the engine for marginal gain, and `tax_rates.tax_percentage` would need the same split.

## C.3 Genuinely missing structures

**None for the core requirements.** Every configuration concept (country catalog, hotel activation, effective dates, hotel rate override, historical tax breakdown) has a home. The only genuinely additive work is the linkage columns on `folio_tax_details` (C.2.1) and Prisma models for two migration-only tables (TX-08, TX-09).

## C.4 Unresolved business decisions

1. **TX-11 — One engine or two?** `tax_groups` (cashiering) and `tax_codes`+overlays (`TaxService`) are independent per-hotel stores with different semantics (always-cumulative vs SUBTOTAL/CUMULATIVE). Unifying them is an architecture decision, not a schema necessity. Options:
   - (a) Keep both; document `TaxService` as authoritative for room/checkout/CRS and `tax_groups` for POS/cashiering transaction codes.
   - (b) Retire `tax_groups` and route cashiering through `TaxService` — requires mapping `transaction_codes.tax_group_id` to `tax_codes`.
   - **Not decided here.** Do not invent the business rule.
2. **TX-13 — Fixed-amount tax periodicity.** `$5/night` needs a unit (night / person / stay / per-die). The schema has no unit column. Accepting the current "flat amount applied once per computation" is the minimal path; a `unit` column is only needed if per-night application is required.
3. **TX-10 — Retire or repair `room_types.tax_rate`.** The column is absent from Prisma and already retired from check-out by test T11. The group-allotment reconcile handler still uses it. Repair (repoint to `TaxService`) is the consistent choice, but this is an application change to a Phase-6-owned handler.
4. **Effective-date granularity.** `hotel_tax_config.effective_from/_to` are `date`. Business date for tax is typically the stay/charge date. Confirm whether the charge date, stay date, or posting date drives effective-date selection.

---

# D. Migration Design Plan (DESIGN ONLY — not created, not executed)

Dependency order:

**Migration 1 — `folio_tax_details` charge linkage (C.2.1)**
1. `ALTER TABLE folio_tax_details ADD COLUMN charge_trx_no integer NULL;`
2. `ALTER TABLE folio_tax_details ADD COLUMN posting_id text NULL;`
3. `ALTER TABLE folio_tax_details ADD COLUMN currency varchar(3) NULL DEFAULT 'USD';`
4. `ALTER TABLE folio_tax_details ADD COLUMN tax_code varchar(20) NULL;`
5. `CREATE INDEX idx_folio_tax_details_charge ON folio_tax_details (hotel_id, charge_trx_no);`
6. `CREATE INDEX idx_folio_tax_details_posting ON folio_tax_details (hotel_id, posting_id);`
7. `ALTER TABLE folio_tax_details ADD CONSTRAINT fk_folio_tax_details_charge FOREIGN KEY (charge_trx_no) REFERENCES folio_charges(trx_no) ON DELETE SET NULL;`
8. `ALTER TABLE folio_tax_details ADD CONSTRAINT fk_folio_tax_details_posting FOREIGN KEY (posting_id) REFERENCES folio_postings(id) ON DELETE SET NULL;`
9. Optional CHECK: `CHECK (charge_trx_no IS NOT NULL OR posting_id IS NOT NULL OR created_at < '<migration_ts>')` to grandfather legacy rows.

**Compatibility:** All columns nullable; no rewrite of existing rows; existing `(hotel_id, folio_id)` index preserved. Application can deploy before or after; old rows remain readable as folio-level summaries.

**Rollback:** Drop the four columns and two indexes and two FKs. No data loss beyond the new (empty) linkage.

**Migration 2 — Prisma models only (no DDL)**
- Add `folio_postings` and `trx_code_config` Prisma models matching existing DDL. No SQL executed; pure client-generation change.

**Migration 3 (conditional, pending TX-11 decision)**
- Only if the business elects to unify engines. Not designed here.

**Data-integrity checks (post-migration, on an authorized isolated DB only):**
- `SELECT count(*) FROM folio_tax_details WHERE charge_trx_no IS NOT NULL AND charge_trx_no NOT IN (SELECT trx_no FROM folio_charges);` → expect 0.
- `SELECT count(*) FROM folio_tax_details WHERE posting_id IS NOT NULL AND posting_id NOT IN (SELECT id FROM folio_postings);` → expect 0.
- Orphan legacy rows (`charge_trx_no IS NULL AND posting_id IS NULL`) expected and acceptable.

---

# E. Verification and Acceptance Plan

## E.1 Repository-only (no DB required)

| # | Check | Evidence method |
|---|-------|-----------------|
| E1 | `getActiveTaxCodes` reads `hotel_tax_config` + `tax_rates` | Unit test with mocked Prisma asserting overlay queries |
| E2 | Empty `hotel_tax_config` → falls back to full country catalog | Unit test: overlay empty → country rows returned |
| E3 | `hotel_tax_config` present → only active + effective codes returned | Unit test with `effective_from`/`effective_to` boundary dates |
| E4 | `tax_rates` override applied within `[begin_date, end_date]`, not outside | Unit test with three date cases |
| E5 | `invalidateCache` called after settings write | Unit test spying `invalidateCache` |
| E6 | `recordFolioTaxDetails` no longer deletes sibling charges' rows | Unit test: two charges → both breakdowns persist |
| E7 | `reconcile-room-tax` no longer references `room_types.tax_rate` | Source-regex test mirroring `check-out.handler.spec.ts:498-503` |
| E8 | UI loads/saves via API, not hardcoded const | Component/integration test with mocked API |
| E9 | Rounding: single policy across engines | Unit test asserting `round2` vs `Math.round` equivalence on edge cases |

## E.2 Requires a separately authorized, verified isolated test database

| # | Check |
|---|-------|
| E10 | Configuration persistence round-trip: `PUT` → `GET` reflects `hotel_tax_config`/`tax_rates` rows |
| E11 | Hotel-specific settings vs country defaults: two hotels in one country, different activations, distinct `computeTax` results |
| E12 | Effective-date boundaries: `effective_from`/`effective_to` inclusive/exclusive behavior against a known business date |
| E13 | Multiple tax components + rounding: stacked SUBTOTAL + CUMULATIVE taxes, verify each `taxable_amount` and `tax_amount` |
| E14 | Multiple charges retain independent tax details on one folio |
| E15 | Idempotent posting: replay with same `idempotencyKey` → no duplicate charges, no duplicate tax details |
| E16 | Concurrent posting: two parallel `postCharge` on same folio → advisory lock serializes; no double `trx_no` |
| E17 | Credit/refund: `postCredit` recomputes without destroying charge-level detail |
| E18 | Historical tax immutability: change a rate after checkout; existing `folio_tax_details` and `folio_charges.trx_amount` unchanged |
| E19 | Reconciliation: `SUM(folio_tax_details.tax_amount)` per charge equals `trx_amount - net` for taxed charges; `folio_header.balance` consistent |
| E20 | RLS on `hotel_tax_config`: verify whether the application role can read/write (baseline enabled RLS; no policy in repo) |

**Safety gate:** E10–E20 must not run against `xylo_cloud`. Target and isolation must be verified and separately authorized first.

---

## 7. Authorization Dependencies (recorded, out of DB-contract scope)

Per instruction, RBAC is **not** redesigned here. Recorded for dependency tracking only:

- `GET front-office/tax-config` has no permission decorator; controller is `@PropertyScope(false)` (`front-office.controller.ts:21,146`).
- Production `AuthService.validateUser` (`auth.service.ts:129-141`) does not populate `request.user.permissions`.
- `public.permissions` / `role_permissions` / `user_roles` have zero production readers; `users.id text` vs `user_roles.user_id integer` are type-incompatible (baseline `:18683` vs `:18656`).
- The working permission path (`platform/permission/`) and `PropertyScopeResolver` depend on `xylo_platform` — **OUT OF SCOPE — REQUIRES EXPLICIT AUTHORIZATION.**

**Database contract compatibility does not make the tax endpoints secure.** TX-17 remains BLOCKED pending an explicit authorization decision.

---

# 9. Final Decision Gate

| # | Question | Answer | Classification |
|---|----------|--------|----------------|
| 1 | **What is already structurally adequate?** | `tax_codes` (country catalog), `hotel_tax_config` (activation + effective dates), `tax_rates` (date-scoped overrides), `hotels.country`, `folio_header`/`folio_charges`/`folio_postings` ledger. The intended architecture is already encoded in migration `20260607003000`. | **VERIFIED** |
| 2 | **What is missing from the database contract?** | Charge/posting-level linkage on `folio_tax_details` (TX-05/06). Currency on tax detail rows (TX-15). Prisma models for `folio_postings` and `trx_code_config` (TX-08/09). Nothing else is genuinely absent. | **VERIFIED** |
| 3 | **What exists but is disconnected from application code?** | `hotel_tax_config` and `tax_rates` — zero production readers/writers. `TaxService.invalidateCache` — zero callers. The Taxes UI — mock with no HTTP. | **DEFECT CONFIRMED** |
| 4 | **What requires application-only changes, not schema changes?** | TX-01/02/03 (wire overlays into `getActiveTaxCodes`), TX-04 (UI wiring), TX-12 (call `invalidateCache`), TX-10 (repoint group reconcile to `TaxService`). | **PROPOSED** |
| 5 | **Minimal proposed schema and migration scope?** | One additive migration adding 4 nullable columns + 2 indexes + 2 FKs to `folio_tax_details`. One model-only migration adding `folio_postings` + `trx_code_config` to Prisma. No new tables. No destructive changes. | **PROPOSED** |
| 6 | **Unresolved business/authorization decisions?** | TX-11 (one tax engine or two — `tax_groups` vs `TaxService`); TX-13 (flat-tax periodicity unit); TX-10 owner (group-allotment vs front-office); effective-date driver (charge/stay/posting date); TX-17 (authorization — BLOCKED on `xylo_platform` scope). | **BLOCKED / NOT VERIFIED** |
| 7 | **Recommended next implementation step, in dependency order?** | (1) Wire `TaxService.getActiveTaxCodes` to `hotel_tax_config` + `tax_rates` with country-catalog fallback (TX-02/03/14) — pure application change, unblocks everything. (2) Add settings read/write endpoints + `invalidateCache` (TX-01/12). (3) Replace UI mock (TX-04). (4) Add `folio_tax_details` linkage migration + fix `recordFolioTaxDetails` (TX-05/06). (5) Repoint `reconcile-room-tax` (TX-10). (6) Add Prisma models (TX-08/09). (7) Resolve TX-11 before any cashiering unification. | **PROPOSED** |

**Live-schema verification: NOT PERFORMED for every table above.** No claim is made that repository definitions match the live `xylo_cloud` schema.

**Phase 3 is not declared complete.** This document is analysis and design only; no code, schema, or migration was created, modified, or executed.

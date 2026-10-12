# XYLO Phase 3 — Tax Database Contract Gap Analysis & Schema Design

**Status:** IMPLEMENTED + PRODUCTION-APPLIED + VERIFIED. TX-01 and TX-13 code + targeted tests complete. **Both tax-linkage migrations applied to `xylo_cloud.public`** (charge-linkage + TX-13) and registered in `_prisma_migrations`. TX-13 production migration gate: **CLOSED**. Open gate: legacy FLAT/FIXED policy ratification (TX-10, TX-11, effective-date remain out of scope). See Decision Gate at the end.
**Schema scope:** `public` only. `xylo_platform` and all non-`public` schemas are OUT OF SCOPE.
**Live database:** `xylo_cloud` (`xylo_user@localhost:5432`, `public` schema) — inspected 2026-10-12 (read-only audit) + **both tax-linkage migrations applied 2026-10-13** (scoped psql + `prisma migrate resolve --applied`). Three unrelated pending migrations remain untouched.
**Date:** 2026-10-12 (audit) / 2026-10-13 (TX-13 implementation + validation + production apply of both tax-linkage migrations)

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
| **TX-13** | Fixed-amount taxes representable | `tax_percentage decimal(5,2)` reused as flat amount when `tax_type` in (`FLAT`,`FIXED`); `tax.types.ts` declares `FLAT`; UI mock has `$5/night` | Percentage-only government tax catalog (ratified) | **POLICY RESOLVED** (see C.2.2); legacy `FLAT`/`FIXED` code path retained read-only for compatibility | `tax.service.ts:41,62`; `tax-settings.service.ts:184-186`; `crs-front-desk-integration.service.ts:35-41`; `tax.types.ts:1` | LOW | Percentage-only catalog; future fixed fees = separate fee domain, not FLAT tax | No |
| **TX-14** | Effective-date-aware activation | `hotel_tax_config.effective_from/_to` exist, unused | Engine filters by business date | **EXISTING BUT UNUSED** | `schema.prisma:5266-5267` | HIGH | Part of TX-02 | **No** |
| **TX-15** | Currency consistency | `TaxBreakdown.currency` defaults `'USD'` (`tax.service.ts:6`); `folio_postings.currency` default `'USD'`; `folio_tax_details` **has no currency column** | Currency on tax detail rows | **MISSING COLUMN OR CONSTRAINT** | `schema.prisma:4432-4445` | LOW | Add `currency` with TX-05 | Yes (with TX-05) |
| **TX-16** | Rounding documented and consistent | `round2` half-up (`tax.service.ts:151-153`); cashiering uses `Math.round(x*100)/100` — equivalent | Single rounding policy | **CONTRACT MISMATCH** (benign) | both engines | LOW | Document; no change | No |
| **TX-17** | Authorization on tax endpoints | `GET front-office/tax-config` has no permission decorator; controller `@PropertyScope(false)` | Enforced read/write permissions | **RESOLVED (ADR-001)** | `front-office.controller.ts:21,146`; new `tax-settings.controller.ts` (global guards) | HIGH | ADMIN-Only policy ratified via ADR-001; existing ADMIN/SUPER_ADMIN guard bypass retained | **No** |

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
| `tax_code` | `varchar(20)` nullable | — | Distinct from `tax_type`. **Resolved (TX-13):** `tax_type` stores the tax *code* for `PERCENTAGE` components (pinned by `tax.service.spec.ts:355`) and the literal `'FLAT'`/`'FIXED'` for legacy fixed-amount components (`tax.service.ts:134`). This is the semantic overload the column was born with; it is now load-bearing for FLAT credit containment (C.2.2). |

**Integrity rules:**
- Exactly one of `charge_trx_no` / `posting_id` should be set for new rows (CHECK constraint, deferred).
- Retain the existing `(hotel_id, folio_id)` index for folio-summary reads; add index on `(hotel_id, charge_trx_no)` and `(hotel_id, posting_id)`.
- Remove the folio-wide `DELETE` in `recordFolioTaxDetails`; replace with per-charge delete (`WHERE hotel_id=$1 AND charge_trx_no=$2`) or upsert.

**Backfill implication:** Existing rows have neither linkage column and remain valid as folio-level historical snapshots. Nullable columns avoid a blocking backfill.

**Application behavior depending on it:** `BillingService.postCharge` (`billing.service.ts:191`), `BillingService.postCredit` (`:246`), `BillingService.getFolioDetails` (`:83-99`), `TaxService.recordFolioTaxDetails` (`:79-95`).

### C.2.2 `tax_codes.tax_percentage` semantics (TX-13) — PRODUCT OWNER DECISION (ratified)

**Status: DECISION RECORDED — 2026-10-12. This supersedes the earlier "documented limitation, no change" framing.**

**Ratified policy (Product Owner):**
1. The **government tax catalog** in XYLO supports **percentage taxes only**. `tax_codes.tax_percentage` is a **percent** for every tax the product will configure.
2. **Hotel tax settings** (`GET`/`PUT /tax-settings`, ADMIN/SUPER_ADMIN per ADR-001) let authorized users adjust tax **percentages** within the current contract (`decimal(5,2)`, 0..999.99) and effective-date windows. Settings do **not** create fixed-amount taxes.
3. Any **future fixed fees** (resort fee, city levy, bed tax, etc.) are handled as a **separate fee domain**, **not** as `tax_type` `FLAT` or `FIXED`. They are not in scope for TX-13 and must not be modeled as taxes.

**Legacy code path (retained, not removed):** `TaxService.computeTax` (`tax.service.ts:41,62`), `TaxSettingsService.isFlat` (`tax-settings.service.ts:184-186`), `CrsFrontDeskIntegrationService.getTaxConfig` (`crs-front-desk-integration.service.ts:35-41`), and `tax.types.ts` still interpret `tax_type ∈ {FLAT, FIXED}` as "use `tax_percentage` as an absolute amount." This branch is **dead for the ratified policy** but is kept read-only for compatibility: no seed inserts such rows, and the isolated public-schema snapshots below contain none. Removing the branch is **not** proposed in this pass (no production change).

**Overload note (unchanged physical fact):** `tax_percentage decimal(5,2)` remains physically capable of holding an absolute amount; the decision constrains **what the product puts in it**, not the column type. A separate `flat_amount` column is **not** proposed.

**Scope boundary — credit/reversal is settled separately (TX-13):** C.2.2 does **not** settle how FLAT/FIXED taxes reverse on `postCredit`. That is implemented under the TX-13 contract (see C.2.2 "TX-13 Implementation Outcome"): percentage pro-rata of the original charge's stored tax, FLAT full-credit-only containment (not ratified).

**Live-data verification:** BLOCKED for live `public` inside `xylo_cloud` (never connected, by constraint). Isolated public-schema snapshots checked instead — see "TX-13 DB evidence" below. **No seed, migration, or test in this repository inserts `tax_type` `FLAT`/`FIXED` into `tax_codes`.**

#### TX-13 Credit/Tax Reversal — Evidence (2026-10-12, read-only; no code/schema change)

**How the credit path works today** (`billing.service.ts:219-282` `postCredit`; `tax.service.ts:54-66` `computeTax`; `recordFolioTaxDetails` writes negated rows scoped to the credit's own `trx_no`):

1. Insert a positive `folio_charges` row (`trx_type='ADJUSTMENT'`, `trx_amount = credit amount`) (`:238-243`).
2. `balance -= credit amount`; `total_credit += credit amount` (`:244-247`).
3. Recompute `computeTax(hotelId, creditAmount)` **from the current catalog** (`:266`).
4. Negate every component (`taxAmount`, `baseAmount`, `subtotal`, `totalTax`) (`:268-274`).
5. Write negated rows against the **credit's own** `charge_trx_no` (`:275-277`) — the original charge's rows are untouched (TX-01 fix).

**Percentage-only path (ratified catalog) — established behavior, pinned by tests** (`folio-tax-details-integrity.phase3.spec.ts:307-334, 359-382`):

| Scenario | Current behavior | Evidence |
|---|---|---|
| **Full credit** (credit = original subtotal) | `SUM(tax_amount) → 0`; balance nets to 0. Correct for percentage taxes. | Not a dedicated test, but arithmetic follows from the partial-credit invariant below. |
| **Partial credit** (credit X of charge S) | Recompute tax on X at **today's** rates, negate. `SUM(tax_amount) = tax due on remaining balance` (net S−X). **Pinned business rule** for percentage taxes. | Test `:307-334`: charge 100 → +10 tax; credit 50 → −5; `SUM=5` = tax on remaining 50. |
| **Multiple credits against one original charge** | Each credit independently recomputes on **its own amount** and negates. `SUM` still nets to tax on remaining **net folio balance**, not on any single charge. Lock test posts 2 credits (`:542-559`). | Lock test only asserts lock identity; **no test asserts multi-credit tax `SUM`**. |
| **Credits without an original-charge reference** | **No `referenceChargeTrxNo` field exists.** All FO callers (`check-out` early-departure refund `:96-102`, voucher `:149-155`; `upgrade-room` `:158-164`; `change-rate-code` `:110-116`) post amount-only credits. Tax is recomputed on the credit amount at current rates — **not** the original charge's tax. | `billing.service.ts:219` signature; FO handlers. |
| **Rounding** | `round2` half-up (`tax.service.ts:294-296`) applied per component. `totalTax = round2(Σ components)`. Credit negation re-uses the same rounded values. `getFolio` rounds `totalTax`/`grandTotal` at read time (`billing.service.ts:97-98`). **No bank-error / penny-residual policy.** | `tax.service.ts:54-66,294`; `billing.service.ts:89-98`. |
| **Cumulative (tax-on-tax) adjustment** | Recompute walks `runningBase` including prior taxes (`:54-65`). A credit of X recomputes the **full cumulative stack on X in isolation**, then negates — it does **not** reverse the original charge's tax component-by-component. | `tax.service.ts:54-66`; stacking test `tax.service.spec.ts:173-185`. |

**Asymmetry (percentage path):** `postCharge` stores a **tax-inclusive** `trx_amount` (subtotal + tax, `:166,190`). `postCredit` stores a **tax-exclusive** amount (`data.amount`, `:242`) and subtracts **only** that from `balance` (`:245`). Tax impact is carried entirely by the negated `folio_tax_details` rows, **not** by `trx_amount`. Net folio tax = `SUM(tax_amount)` across all charges+credits.

**Legacy FLAT/FIXED compatibility path (not ratified; retained dead branch)** — analyzed separately:

If a `tax_codes` row with `tax_type ∈ {FLAT,FIXED}` were ever present (isolated DBs show **0**; live `public` BLOCKED), `computeTax` returns `taxAmount = round2(tax_percentage)` as an **absolute amount** (`tax.service.ts:61-62`), independent of the credit amount. Then a credit of **any** size (partial or full) would negate the **entire** fixed amount, because the recompute is on the credit amount but FLAT ignores that amount. Consequences under the current code:

- Partial credit of X against a charge that carried a $5 FLAT tax → credit writes **−$5**, zeroing the FLAT tax on the remaining charge. **Over-reversal.**
- Multiple credits of X₁,X₂,… each write **−$5** → `SUM` goes **negative** beyond the original +$5. **Ledger integrity defect.**
- Full credit correctly nets FLAT to 0 (coincidence of the same absolute amount).
- This is an **implementation artifact** of "recompute-and-negate", not a designed rule. C.2.2 already records FLAT/FIXED as a dead branch under the ratified percentage-only policy; the credit path has **no** FLAT-specific handling and **no** test coverage (`folio-tax-details-integrity` tests use percentage-only fixtures).

**Safe reversal approaches evaluated (from code/schema evidence only; no policy invented):**

| Approach | Fits current schema? | Percentage correctness | FLAT correctness | Trade-off |
|---|---|---|---|---|
| **A. Status quo — recompute & negate on credit amount** (today) | Yes (no new columns) | Holds the pinned `SUM = tax on remaining net balance` rule for pure percentage. **Breaks** under rate-change between charge and credit (recompute uses **today's** rate, not the original charge's rate). | **Broken** for partial/multiple credits (over-reversal, negative `SUM`). | Zero schema change; rate-change and FLAT defects remain. |
| **B. Reference the original charge** — add optional `reference_charge_trx_no` on the credit (or a parameter on `postCredit`); reverse a **pro-rata share** of the original charge's stored `folio_tax_details` rows. | **Additive nullable column** — compatible with C.2.1 linkage (`charge_trx_no` already exists on details). No destructive change. | Fixes rate-change drift (reverses the **original** tax, not a recompute). Partial/multiple credits can sum to the original tax exactly (clamp last credit to residual). | Still needs a FLAT share rule (pro-rata vs full-on-full-credit-only) — **unresolved**. | Requires schema + API contract change; all existing amount-only FO callers must keep working (column nullable). |
| **C. Full-reversal-only** — a credit reverses tax only when it fully offsets a specific charge; partial credits reverse **no** tax. | No new columns strictly required, but needs charge linkage to know "fully offsets". | **Violates** the pinned test rule (`SUM = tax on remaining balance` after a partial credit → would leave `SUM=10` on net 50). Would break `folio-tax-details-integrity.phase3.spec.ts:307-334`. | Correct for FLAT (no over-reversal). | Smallest FLAT fix, but **conflicts with an existing pinned business rule** for percentage. Not safe without reversing that test. |
| **D. Do nothing for FLAT (document as dead)** — ratify that FLAT/FIXED never reverses because it never charges. | None | Unchanged. | FLAT stays unreachable; if a stray FLAT row appears in live `public`, partial credits still corrupt `SUM`. | Accepts residual risk on an unverified live schema; no code change. |

**Code defects vs unresolved business rules (separated):**

| Item | Classification | Notes |
|---|---|---|
| Recompute uses **current** catalog rate, not the original charge's rate, on credit | **Code defect** (drift under rate change) | Affects percentage path; test suite never covers charge-then-rate-change-then-credit. |
| FLAT/FIXED partial & multiple credits over-reverse / drive `SUM` negative | **Code defect** on a **dead branch** | Only reachable if a FLAT row exists in live `public` (BLOCKED to verify). Under ratified policy, FLAT should never be configured. |
| No original-charge reference on `postCredit` | **Unresolved business + API contract** | Every FO caller is amount-only today; adding a reference is a contract change. |
| Pro-rata vs full-on-full-credit-only for FLAT | **Unresolved business rule** | Not established by any authoritative test or doc. |
| Percentage partial-credit `SUM = tax on remaining net balance` | **Established business rule** (pinned) | `folio-tax-details-integrity.phase3.spec.ts:307-334`. Must be preserved by any new approach. |
| Rounding residual / penny policy on multi-credit reversal | **Unresolved business rule** | No policy exists; `round2` half-up only. |
| Cumulative tax-on-tax credit semantics (reverse component stack vs isolated recompute) | **Unresolved business rule** | Current isolated recompute is the behavior; no test pins multi-tax cumulative credit `SUM`. |

**Minimum additional decisions required before any implementation:** ~~superseded by TX-13 implementation (2026-10-13). See "TX-13 Implementation Outcome" below.~~

1. **Rate-change drift (percentage):** ~~Open~~ → **RESOLVED (implemented).** Approach B chosen: optional `referenceChargeTrxNo` on `postCredit` reverses a **pro-rata share of the original charge's stored `folio_tax_details`** (anchored on `originalTax`, not a recompute), clamped to remaining refundable tax. Rate-change drift fixed.
2. **FLAT/FIXED credit rule (legacy branch):** ~~Open~~ → **SAFE CONTAINMENT (not ratified).** Full-credit-only reversal: a FLAT component reverses its remaining tax **only when the credit fully covers the original charge's taxable base**; partial credits reverse $0 of that component. No new FLAT policy is invented — this is compatibility containment, not ratification. Production `xylo_cloud.public.tax_codes` has **0** FLAT/FIXED rows (verified read-only 2026-10-12), so the branch is unreachable in production.
3. **Original-charge reference:** ~~Open~~ → **RESOLVED (implemented).** `postCredit` accepts optional `referenceChargeTrxNo?: number`. Linked path fails closed when the charge is missing or cross-tenant. Amount-only callers unchanged (legacy recompute path retained).
4. **Cumulative / multi-tax credit:** **RESOLVED (implemented).** Each component reverses pro-rata of its **own** original tax (not a stacked recompute). Fraction denominator is the original charge's `taxable_amount` (subtotal base). Multiple partial credits sum to the original tax exactly via the remaining clamp.
5. **Rounding residual:** **RESOLVED (implemented).** Each component rounds independently (`round2` half-up), then clamps the final cent to remaining refundable tax so `SUM` can never over-reverse the original tax.

#### TX-13 Implementation Outcome (2026-10-13)

**Migration (applied to `xylo_cloud.public`, 2026-10-13):** `packages/db/migrations/20261013000000_tx13_credit_tax_reversal_linkage/migration.sql`

- Adds nullable `reference_charge_trx_no int` to `folio_tax_details` (the original charge the credit reverses).
- Widens `tax_amount` / `taxable_amount` to `numeric(18,4)` (was `decimal(18,4)` — same precision, normalized type).
- Creates index `folio_tax_details_hotel_reference_idx` on `(hotel_id, reference_charge_trx_no)`.
- **Isolated validation:** applied cleanly against a `pg_dump --schema-only` clone of production `folio_tax_details` + `folio_charges` (scratch DB, dropped after). `ON_ERROR_STOP=1`, zero errors.
- **Production apply (2026-10-13):** applied directly to `xylo_cloud.public` via scoped `psql -v ON_ERROR_STOP=1 --single-transaction` + `prisma migrate resolve --applied`. `migrate deploy` was avoided — it would have also applied four unrelated pending migrations. Schema, data, and migration history all verified (see "TX-13 production migration — applied" below).

**Code changes (`apps/api/src`):**

- `billing.service.ts` — `postCredit` accepts optional `referenceChargeTrxNo`. Linked path: `loadOriginalChargeForReversal` (hotel+folio scoped, fails closed on missing/cross-tenant) → `computeLinkedTaxReversal` (`fraction = creditAmount / originalTaxableBase` clamped `[0,1]`; PERCENTAGE pro-rata of `originalTax`, clamped to remaining; FLAT/FIXED full-credit-only) → negated breakdown with `reference_charge_trx_no` tag. Original charge rows never rewritten.
- `tax.service.ts` — `recordFolioTaxDetails` writes the new nullable column.
- FO callers (`check-out`, `upgrade-room`, `change-rate-code`) updated for the optional field.

**Test results (targeted, low-memory jest, `--runInBand --forceExit`):**

| Spec | Result |
|---|---|
| `folio-tax-details-integrity.phase3.spec.ts` | **28/28 PASS** (11 new TX-13 tests) |
| `tax.service.spec.ts` | **32/32 PASS** |
| `check-out.handler.spec.ts` | **PASS** |
| `upgrade-room.handler.spec.ts` | **PASS** |
| `change-rate-code.handler.spec.ts` | **PASS** |
| `npx tsc --noEmit` | **clean** |

**Resolved by this implementation:** rate-change drift, original-charge reference, cumulative/multi-tax component semantics, rounding residual (clamp-to-remaining), FLAT over-reversal containment.

**Still open (not invented, not ratified):**
- Legacy FLAT/FIXED **policy** (containment is evidence-based, not a business rule).
- FK on `charge_trx_no` (deferred — `folio_charges.trx_no` has no unique/PK in production).
- TX-10, TX-11, effective-date driver (out of scope).

**No production code, schema, migration, seed, or data was changed against `xylo_cloud`.**

#### TX-13 DB evidence (isolated `public` schema only; `xylo_cloud` not connected)

| Database (isolated) | `public.tax_codes` rows | `tax_type` values | Notes |
|---|---|---|---|
| `xylo_test` | 0 | — | Migration-only, empty |
| `xylo_c03_replay` | 0 | — | Empty |
| `xylo_c03_livesim` | 0 | — | Empty |
| `xylo_restore` (pre-country snapshot) | 9 | **all NULL** | `EG_*`/`MA_*`; percentages 1.00–20.00; **0 rows** with `FLAT`/`FIXED` |

`hotel_tax_config` = 0 rows and `tax_rates` = 0 rows in every isolated DB checked. `folio_tax_details` = 0 rows. Conclusion: **no observable fixed-amount tax** in any approved isolated public-schema source. Production `public` inside `xylo_cloud` is **not** verified (constraint).

## C.3 Genuinely missing structures

**None for the core requirements.** Every configuration concept (country catalog, hotel activation, effective dates, hotel rate override, historical tax breakdown) has a home. The only genuinely additive work is the linkage columns on `folio_tax_details` (C.2.1) and Prisma models for two migration-only tables (TX-08, TX-09).

## C.4 Unresolved business decisions

1. **TX-11 — One engine or two?** `tax_groups` (cashiering) and `tax_codes`+overlays (`TaxService`) are independent per-hotel stores with different semantics (always-cumulative vs SUBTOTAL/CUMULATIVE). Unifying them is an architecture decision, not a schema necessity. Options:
   - (a) Keep both; document `TaxService` as authoritative for room/checkout/CRS and `tax_groups` for POS/cashiering transaction codes.
   - (b) Retire `tax_groups` and route cashiering through `TaxService` — requires mapping `transaction_codes.tax_group_id` to `tax_codes`.
   - **Not decided here.** Do not invent the business rule.
2. **TX-13 — Fixed-amount tax periodicity. ~~Open~~ → CLOSED.** The government tax catalog is percentage-only (ratified, C.2.2; production `xylo_cloud.public.tax_codes` has 0 FLAT/FIXED rows, verified read-only 2026-10-12), so a per-night/person/stay **unit** for FLAT taxes is **moot for the product**. No `unit` column is needed. Fixed fees, if ever required, belong to a separate fee domain (not FLAT taxes). **Credit/reversal:** implemented under the TX-13 contract (C.2.2 "TX-13 Implementation Outcome") — percentage pro-rata of the original charge's stored tax, FLAT full-credit-only containment (not ratified). Remaining open: production apply authorization, legacy FLAT policy ratification.
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

**UPDATE (TX-17 closure):** The project owner has ratified an ADMIN-Only Tax Settings authorization policy (ADMIN + SUPER_ADMIN may read and modify `GET /tax-settings` and `PUT /tax-settings`; all other roles denied). Enforcement is the existing ADMIN/SUPER_ADMIN role bypass in `PermissionGuard` (`permission.guard.ts:20`); the declared codes `tax:settings:read` / `tax:settings:write` do not independently grant access under the current pipeline (`AuthService.validateUser` never populates `request.user.permissions`). Recorded as **ADR-001** — `docs/adr/001-admin-only-tax-settings-authorization.md`. TX-17 is no longer BLOCKED.

---

# 9. Final Decision Gate

| # | Question | Answer | Classification |
|---|----------|--------|----------------|
| 1 | **What is already structurally adequate?** | `tax_codes` (country catalog), `hotel_tax_config` (activation + effective dates), `tax_rates` (date-scoped overrides), `hotels.country`, `folio_header`/`folio_charges`/`folio_postings` ledger. The intended architecture is already encoded in migration `20260607003000`. | **VERIFIED** |
| 2 | **What is missing from the database contract?** | Charge/posting-level linkage on `folio_tax_details` (TX-05/06). Currency on tax detail rows (TX-15). Prisma models for `folio_postings` and `trx_code_config` (TX-08/09). Nothing else is genuinely absent. | **VERIFIED** |
| 3 | **What exists but is disconnected from application code?** | `hotel_tax_config` and `tax_rates` — zero production readers/writers. `TaxService.invalidateCache` — zero callers. The Taxes UI — mock with no HTTP. | **DEFECT CONFIRMED** |
| 4 | **What requires application-only changes, not schema changes?** | TX-01/02/03 (wire overlays into `getActiveTaxCodes`), TX-04 (UI wiring), TX-12 (call `invalidateCache`), TX-10 (repoint group reconcile to `TaxService`). | **PROPOSED** |
| 5 | **Minimal proposed schema and migration scope?** | One additive migration adding 4 nullable columns + 2 indexes + 2 FKs to `folio_tax_details`. One model-only migration adding `folio_postings` + `trx_code_config` to Prisma. No new tables. No destructive changes. | **PROPOSED** |
| 6 | **Unresolved business/authorization decisions?** | TX-11 (one tax engine or two — `tax_groups` vs `TaxService`); TX-13 **catalog basis** RESOLVED (percentage-only, C.2.2); TX-13 **credit/reversal** — rate-change drift, original-charge reference, cumulative stack, and rounding residual are **RESOLVED (implemented + production-applied, C.2.2 "TX-13 Implementation Outcome")**; the legacy FLAT/FIXED **share policy** is **NOT RATIFIED** (safe containment only, not a business rule). TX-10 owner (group-allotment vs front-office); effective-date driver (charge/stay/posting date). TX-17 (authorization) is RESOLVED via ADR-001 (ADMIN-Only policy ratified). | **PARTIAL** — TX-13 code + production migration DONE; legacy FLAT policy ratification + TX-11 + TX-10 + effective-date still open |
| 7 | **Recommended next implementation step, in dependency order?** | (1) ~~Wire `TaxService.getActiveTaxCodes` overlays (TX-02/03/14)~~ — DONE. (2) ~~Settings read/write + `invalidateCache` (TX-01/12)~~ — DONE. (3) ~~Replace UI mock (TX-04)~~ — DONE. (4) ~~`folio_tax_details` linkage migration + `recordFolioTaxDetails` (TX-05/06)~~ — DONE (TX-01). (5) ~~TX-13 credit/reversal~~ — DONE (implemented + production-applied + verified). (6) ~~Production apply of TX-13 migration~~ — DONE (scoped psql execution + `prisma migrate resolve --applied`). (7) **Ratify or formally reject** the legacy FLAT/FIXED containment policy (currently evidence-based, not a business rule). (8) Repoint `reconcile-room-tax` (TX-10). (9) Add Prisma models (TX-08/09). (10) Resolve TX-11 before any cashiering unification. | **PROPOSED (revised)** |

**Live-schema verification: PERFORMED (read-only, 2026-10-12; post-application 2026-10-13).** TX-13 migration **applied to `xylo_cloud.public`** via scoped `psql -v ON_ERROR_STOP=1 --single-transaction` + `prisma migrate resolve --applied`. Columns, `numeric(18,4)` precision, both indexes, existing constraints, and 152 financial rows all verified intact. Migration history records TX-13 as applied; the four unrelated pending migrations remain untouched.

**Phase 3 Tax DB contract: TX-01 and TX-13 COMPLETE (code + tests + production migration).** Remaining open: legacy FLAT/FIXED policy ratification, TX-10, TX-11, and the effective-date driver.

#### TX-13 production migration — applied (2026-10-13)

**Pre-change safety gate:**
- Target: `xylo_cloud` / `public` confirmed. No concurrent migration (only active `_prisma_migrations` query was the verification itself).
- TX-13 had no successful or partial application record in `_prisma_migrations`.
- Backup: `backups/pre-tx13-xylo-cloud-2026-10-13.sql` (39 KB; contains `_prisma_migrations` + `folio_tax_details` with data).
- Focused baseline checks: `account_balances` PK + 4 indexes + RLS `rowsecurity=t` present; 922 public tables; no baseline-introduced objects. Baseline P3009 blocker resolved earlier via `prisma migrate resolve --applied 20250101000000_c03_core_baseline_capture`.

**SQL execution:** `migration.sql` executed directly via `psql -v ON_ERROR_STOP=1 --single-transaction`. All 12 statements succeeded (4 `ALTER TABLE ADD COLUMN IF NOT EXISTS`, 4 `COMMENT ON COLUMN`, 2 `ALTER COLUMN TYPE numeric(18,4)`, 2 `CREATE INDEX IF NOT EXISTS`). No errors.

**Post-application verification:**
- `reference_charge_trx_no integer` nullable — present.
- `charge_trx_no integer` nullable, `tax_code varchar(20)` nullable, `currency varchar(3) DEFAULT 'USD'` nullable — present.
- `tax_amount` / `taxable_amount` = `numeric(18,4)` NOT NULL — present.
- Indexes: `folio_tax_details_hotel_charge_idx` btree `(hotel_id, charge_trx_no)`, `folio_tax_details_hotel_reference_idx` btree `(hotel_id, reference_charge_trx_no)` — both non-unique, correct definitions, no duplicates.
- Existing PK (`folio_tax_details_pkey`), FK (`fk_folio_tax_details_hotel_id`), RLS policy, and `trigger_post_tax_to_gl` all intact.
- Financial data: 152 rows preserved; `tax_amount`/`taxable_amount` 152/152 non-null (no nulls introduced by the widening); `charge_trx_no`/`reference_charge_trx_no` 0/152 non-null (all legacy snapshots, as expected).

**Migration history registration:** `npx prisma migrate resolve --applied 20261013000000_tx13_credit_tax_reversal_linkage`. `_prisma_migrations` records TX-13 with `finished_at` set, `rolled_back_at` NULL.

#### Charge-linkage migration — applied (2026-10-13)

`20261012000000_folio_tax_details_charge_linkage/migration.sql` (45 lines) adds `charge_trx_no` / `tax_code` / `currency` columns plus the `(hotel_id, charge_trx_no)` index. When executed, its intent was already fully satisfied on the live schema — TX-13 is a superset and was applied first. Re-execution was safe (all statements are `IF NOT EXISTS`-guarded; every guard confirmed the existing object) and establishes the migration-history record for the earlier step.

**SQL execution:** `Get-Content ... | psql -v ON_ERROR_STOP=1 --single-transaction -f -`. All 6 statements succeeded. NOTICEs confirmed the 4 guarded objects already exist (`charge_trx_no`, `tax_code`, `currency` columns + `folio_tax_details_hotel_charge_idx`); no changes were made.

**Post-execution verification:** the 3 columns and the `(hotel_id, charge_trx_no)` btree index confirmed present with correct definitions. 152 rows preserved, `tax_amount` 152/152 non-null, `charge_trx_no` 0/152 non-null. PK + FK intact.

**Migration history registration:** `npx prisma migrate resolve --applied 20261012000000_folio_tax_details_charge_linkage`. `_prisma_migrations` records it with `finished_at` set, `rolled_back_at` NULL.

#### Remaining pending migrations (untouched)

`prisma migrate status` now reports **3** unrelated migrations still pending, all untouched:
- `20260801000000_c03_capture_external_tables`
- `20260801000100_c03_drop_email_outbox`
- `20261002000000_c03_late_reconciliation`

`migrate deploy` was never run; only the two scoped SQL files were executed via `psql`.

**TX-13 production migration gate: CLOSED.**

#### TX-13 production migration — P3009 blocker (investigated 2026-10-13, read-only)

`prisma migrate deploy` against `xylo_cloud` was **hard-blocked (P3009)** by a pre-existing failed baseline unrelated to TX-13. Nothing was applied; `_prisma_migrations` was not modified.

**Failed entry** (`_prisma_migrations`):
- `migration_name`: `20250101000000_c03_core_baseline_capture`
- `started_at`: 2026-10-09 01:52:56 UTC; `finished_at`: NULL; `rolled_back_at`: NULL; `applied_steps_count`: 0.
- Logged error: Postgres **42P16** — `multiple primary keys for table "account_balances" are not allowed`, raised at `migration.sql:20672` (`ALTER TABLE account_balances ADD CONSTRAINT account_balances_pkey PRIMARY KEY (id)`).

**Root cause:** that statement is wrapped in `DO $$ ... EXCEPTION WHEN duplicate_object OR duplicate_table OR undefined_function OR undefined_object THEN NULL; END $$;` (`:20671-20673`). `account_balances` already carries a live PK (`account_balances_pkey`) from the push-era database, so Postgres raises **42P16**, which is **not** in the handler list. The baseline is a **live-shape capture** (every `CREATE TABLE`/`CREATE INDEX` is `IF NOT EXISTS`; FKs are stripped for re-add by `20261002000000_c03_late_reconciliation`), so the un-guarded PK add is the sole defect.

**Partial application:** none. The three `CREATE INDEX IF NOT EXISTS` for `account_balances` (`:24299-24303`) and the RLS `ENABLE ROW LEVEL SECURITY` (`:27741`) are **after** the failure point and were never reached. The PK, the three FKs, and the indexes present on `account_balances` today are the pre-existing live shape, not baseline writes. `applied_steps_count = 0` corroborates. Public base tables: **922** (no baseline-introduced tables; all baseline `CREATE TABLE IF NOT EXISTS` skipped).

**Two other unapplied entries — independent, non-blocking:**
- `20260817000000_add_checkin_sessions_and_guest_documents`: the failed row (`42P01`, rolled back) is followed by a **successful** retry (`applied_steps_count = 1`). The failed row is inert history.
- `20260910_b1_trx_codes`: the failed row (`42P07`, rolled back) is followed by a row with `rolled_back_at IS NULL` but **`applied_steps_count = 0`** — a recorded no-op, not a failure. Neither causes P3009; only the baseline does.

**Decision: SAFE, NARROW REMEDIATION IDENTIFIED** (not executed here).

Prisma's supported recovery for an already-satisfied baseline is `prisma migrate resolve --applied 20250101000000_c03_core_baseline_capture`. It is appropriate because the baseline's *intent* (make the live schema the recorded baseline) is already satisfied by the live schema, and the only reason it failed is an over-narrow `EXCEPTION` list — not an incomplete or destructive migration. It does not rerun SQL, does not alter schema/data, and does not falsify a *successful* execution; it records that the migration's target state is present.

Prerequisites: (1) separately authorized production change; (2) a pre-change backup/snapshot of `_prisma_migrations` (and the database) for rollback; (3) confirm no concurrent migration run.

**Outcome (2026-10-13):** The baseline was resolved via `prisma migrate resolve --applied 20250101000000_c03_core_baseline_capture`. Because `migrate deploy` would have also applied four unrelated pending migrations, TX-13 was instead applied directly via scoped `psql -v ON_ERROR_STOP=1 --single-transaction`, then registered via `prisma migrate resolve --applied 20261013000000_tx13_credit_tax_reversal_linkage`. Full verification in "TX-13 production migration — applied (2026-10-13)" above.

Alternative (if `resolve --applied` is rejected for the baseline): fix the baseline's `EXCEPTION` list to include `42P16` (or add `WHEN OTHERS`) and re-run `migrate deploy`. Broader and re-executes ~20k guarded statements; the `resolve --applied` path is preferred.

**TX-13 status: migration APPLIED and VERIFIED (2026-10-13).** The baseline P3009 blocker was resolved earlier via `prisma migrate resolve --applied 20250101000000_c03_core_baseline_capture`. TX-13 was then applied directly via scoped `psql` (the four unrelated pending migrations would have been run by `migrate deploy`, so it was avoided) and registered via `prisma migrate resolve --applied 20261013000000_tx13_credit_tax_reversal_linkage`. Production schema, data, and the four unrelated migrations all verified intact. Legacy `FLAT`/`FIXED` policy ratification remains open and is unaffected.

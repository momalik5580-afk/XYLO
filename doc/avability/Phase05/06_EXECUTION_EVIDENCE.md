# XYLO Availability — Phase 5 Execution Evidence (Stage 6)

**Status:** living evidence record required by `04_IMPLEMENTATION_PLAN.md` §10/§13 (evidence tasks only — never a new audit/spec/plan).
**Environment:** local workspace only; DB = `xylo-cloud` on `xylo-postgres` (read-only queries; `AVAILABILITY_TEST_DATABASE_URL` harness for T5-06).
**Authority:** `04_IMPLEMENTATION_PLAN.md` (execution), `03_FINAL_DOMAIN_SPECIFICATION.md` (domain), `05_READINESS_REVIEW.md` (gates).
**Rule:** G-11 — evidence is observed or recorded OPEN; never fabricated.

---

## E-1 — Population Audit (T5-05) `[GATE: BLK-P5-01 input]`

**Executed:** Stage 6, read-only `information_schema` + `COUNT`/`MAX` queries + repo writer search (rg, specs excluded).
**Scope:** 10 primary stores + 2 excluded stores (per plan §10).

### Store classifications (Acceptance: writer+populated / writer+empty / writer UNKNOWN / population UNKNOWN)

| # | Store | Rows | Per-hotel | Recency (max created/updated or date col) | Writer | Classification |
|---|-------|------|-----------|-------------------------------------------|--------|----------------|
| 1 | `room_inventory` | 0 | — | — | **UNKNOWN** (no raw/Prisma writer in repo) | `writer UNKNOWN` (population = 0 rows) |
| 2 | `out_of_order` | 0 | — | — | **UNKNOWN** (no raw/Prisma writer; reader: `availability-source.adapter.ts:75`) | `writer UNKNOWN` (population = 0 rows) |
| 3 | `out_of_service` | 0 | — | — | **UNKNOWN** (no raw/Prisma writer; reader: `availability-source.adapter.ts:76`) | `writer UNKNOWN` (population = 0 rows) |
| 4 | `rate_restrictions` | 16 | HRG=16 | 2026-08-15 14:16:18 | **UNKNOWN** (no writer found; readers: `crs-engine.service.ts:152,372`) | `writer UNKNOWN` + populated |
| 5 | `close_to_arrival` | 6 | HRG=6 | 2026-09-17 14:45:37 | **IDENTIFIED** — A3 `availability-sales.controller.ts:677,853` | `writer identified + populated` |
| 6 | `close_to_departure` | 1 | HRG=1 | 2026-08-15 16:55:56 | **IDENTIFIED** — A3 `availability-sales.controller.ts:687,862` | `writer identified + populated` |
| 7 | `zero_sell_limits` | 0 | — | — | **IDENTIFIED** — A3 `availability-sales.controller.ts:698,703,871` | `writer identified + empty` |
| 8 | `room_type_sell_limits` | 0 | — | — | **IDENTIFIED** — A3 `availability-sales.controller.ts:826` | `writer identified + empty` |
| 9 | `minimum_length_of_stay` | 0 | — | — | **IDENTIFIED** — A3 `availability-sales.controller.ts:835` | `writer identified + empty` |
| 10 | `maximum_length_of_stay` | 0 | — | — | **IDENTIFIED** — A3 `availability-sales.controller.ts:844` | `writer identified + empty` |
| — | **excluded** `restrictions` (CUTOFF) | 207 | HRG=69, MAK=69, RAK=69 | 2026-05-26 18:34:55 | **IDENTIFIED** — A3 `availability-sales.controller.ts:878-885` | documented excluded (T5-47 overlap) |
| — | **excluded** `channel_restrictions` | 0 | — | — | **UNKNOWN** (no writer) | documented excluded until writer exists (BR-5-046) |

- Population UNKNOWN: **none** (all 12 stores queried live).
- Writer UNKNOWN: `room_inventory`, `out_of_order`, `out_of_service`, `rate_restrictions`, `channel_restrictions` → per BR-5-046 these stay **flagged/excluded, never trusted** in T5-07 scope.
- No store default-assumed. No query mutated data.

### E-2 conflict detection (collected inside T5-05)

| Pair (same hotel + date + key) | Conflicting keys |
|---|---|
| `rate_restrictions` ⋈ `restrictions` (hotel, rate_code, date) | **0** |
| `close_to_arrival` ⋈ `restrictions` (hotel, date) | **0** |
| `close_to_departure` ⋈ `restrictions` (hotel, date) | **0** |
| `close_to_arrival` ⋈ `rate_restrictions` (hotel, date) | **0** |
| `close_to_departure` ⋈ `rate_restrictions` (hotel, date) | **0** |

All other store pairs are non-overlapping (empty stores cannot conflict).

**E-2 result:** **no actual conflicts found ⇒ default stands** (E-2 remains `DEFERRED`; any future conflict ⇒ `UNRESOLVED` + `sourceConflicts[]`, BR-5-003; no precedence ranking invented). No E-2 STOP triggered.

**E-1 status: RECORDED (PASS).** Feeds T5-07 scope and the T5-09 gate.

---

## E-3 — Runtime fail-closed confirmation (T5-06) `[GATE: BLK-P5-01 input]`

**Executed:** Stage 6, safe-environment execution against the **current stub binding** (`availability.module.ts:26` → `UnresolvedRestrictionAdapter`, unchanged), with `AVAILABILITY_TEST_DATABASE_URL` = local `xylo-cloud` (harness schemas isolated + dropped). No code change.

### Observation — four chain links (plan §10: stub → snapshot → assertion → HTTP)

| Link | Execution | Observed evidence | Result |
|---|---|---|---|
| 1. stub | `restriction.adapter.spec.ts` | stub returns `status: UNRESOLVED` + `unresolvedSources[]` (incl. `zero_sell_limits`, `close_to_arrival/departure`, `min/max LOS`, `room_type_sell_limits`, `allotment_stop_sales`); live warn `availability.restriction.source_unavailable` seen at runtime | PASS |
| 2. snapshot | `availability-snapshot.service.spec.ts` | `bookingEligibility === 'UNKNOWN'` (spec :77, :130) + `restrictionOutcome.dimensions.*.status === 'UNRESOLVED'` (:96); real `AvailabilitySnapshotService` ternary path exercised | PASS |
| 3. assertion | `availability-fail-closed.postgres.spec.ts` (FC-01…) | create attempt with unresolved inputs ⇒ `result.status = 'REJECTED'`, `rejection.code = 'UNRESOLVED_CAPACITY'`; **persisted** `availability_assertions.status = 'REJECTED'`; zero balances/movements; reservation state + journal rolled back together | PASS |
| 4. HTTP | `reservation-create-availability.postgres.spec.ts:169` | `AVAILABILITY_ASSERTION_REJECTED` thrown as `AppError` with `statusCode: 409` (+ `rejection` payload) from `reservation-availability-wiring.ts:75-80` | PASS |

**Run:** `jest --runInBand restriction.adapter.spec availability-snapshot.service.spec availability-fail-closed.postgres.spec reservation-create-availability.postgres.spec` → **4/4 suites, 21/21 tests passed** (postgres suites executed live, not skipped).

### Matrix flag OFF (recorded per plan)

- `config.service.ts:63-66`: `FEATURE_GBA_A3_AUTHORITATIVE` (key of `getFeatureFlag('gba.a3.authoritative')`, `availability-sales.controller.ts:275`) defaults to `'false'`.
- Runtime env: `FEATURE_GBA_A3_AUTHORITATIVE` **not present**; `.env` + `compose.yaml` contain zero `FEATURE_` lines (Stage 5 OBS-2, re-confirmed).

**Deviation check:** observed evidence matches static prediction (F-01) on all four links — **no contradiction**.

**E-3 status: RECORDED (PASS).** Together with E-1, this satisfies the evidence inputs of BLK-P5-01 (evaluator binding swap still requires T5-07 + T5-08 green).

---

## E-4 — CUTOFF meaning evidence (T5-87) `[GATE: E-4, non-blocking]`

**Sources searched (read-only, local):** `docs/**` (Phases 3–5, enterprise, design, reservations), `packages/db/schema.prisma` + migrations, DB table/column comments (`pg_description`, `obj_description`), repo code (`'CUTOFF'` writer only at A3 `:878`), live data distributions.

**Findings:**
- `restrictions` table comment: *"Rate and availability restrictions"* — generic; no `rate_code='CUTOFF'` semantics defined anywhere in corpus.
- `zero_sell_limits` table comment: *"Stop-sell indicators"*; `zero_sell_type`/`zero_sell_value` are bare `VARCHAR(20)`, no domain definition, no value distribution (store empty).
- Live data: `restrictions` contains only `rate_code='BAR'` (207 rows) — **zero `CUTOFF` rows exist**; `zero_sell_limits` empty.
- Repo-wide: no reader of `restrictions`; no product doc, glossary, or ops record defines either meaning. (Phase-4 "cutoff" hits are the unrelated `group_blocks.cutoff_date` concept.)

**E-4 result: MEANING UNKNOWN — documented.** Per plan §10 T5-87 failure rule:
- `restrictions (rate_code='CUTOFF')` remains **excluded from authority read/write path forever in this phase** (AC-31, BR-5-024) — no migration attempted.
- Evaluator scope unchanged (FDS amendment would be required to change it).
- `zero_sell_value` domain undefined ⇒ `zero_sell_limits` stays in scope only per its evidenced writer semantics (stop-sell indicator, in-store empty), never trusted beyond flagging (BR-5-046).
- **T5-47 guard remains.**

**E-4 status: RECORDED (documented "meaning unknown").** Non-blocking.

---

## E-5 — External consumer inventory for `/rates/engine/*` (T5-88) `[GATE: E-5, timing only]`

**Sources reviewed (local ops evidence only; no GitHub):**
- `gateway/ingress/nginx.conf`: generic routing only (`/`→web, `/api/`→api, `/admin/`→admin, `/ws/`→ws). **No `/rates/engine`-specific rules, no external upstreams, no partner/IP allowlists.**
- Running containers (`docker ps -a`): temporal + postgres + redis only — **no gateway/nginx container exists ⇒ no gateway access logs to review**; no local access-log files found (`gateway/`, `logs/`, `ops/` empty/absent).
- Repo docs (`docs/**`): every documented `/rates/engine/*` consumer is in-repo (FE `use-crs-book.ts`, `front-office.api.ts:193`, reservations audit endpoint table). No out-of-repo consumer named anywhere.

**E-5 result:** local review executed, but **positive proof of absence is not possible without gateway/ops logs ⇒ external inventory recorded as UNKNOWN (not "none found")**.

Per plan §10 T5-88 failure rule (**inventory unknown** branch):
- In-repo consumer retirements/re-points **proceed** (disposition unchanged — G-4 keeps every rule).
- **External-facing retirement timing waits** — `T5-22…25` route retirements keep revert-safe execution only after an ops log check when logs become available; dispositions unchanged.
- Annotated: T5-22/23 (reads), T5-24 (book), T5-25 (release) = disposition **PROCEED**, timing **awaiting ops-log confirmation**.

**E-5 status: RECORDED (inventory unknown; timing annotated).** Non-blocking for disposition.

---

## E-6 — OTA overbooking branch live check (T5-89) `[GATE: E-6, informative]`

**Executed:** sandbox observation against the **real compiled** `dist/.../webhook-ingress/webhook.service.js` `process()` catch-block (`webhook.service.ts:104`), signature/normalize stubbed, temp script outside repo (`%TEMP%/opencode/e6-webhook-branch.js`). No repo change.

| Obs | Injected failure | Live behavior |
|---|---|---|
| 1 | `Error('insufficient inventory for room type')` (message contains `inventory`) | branch **matches** → WARN `[Webhook] booking-com overbooking: …` → returns `{received:true, duplicate:false}` (swallowed) |
| 2 | Authority contract `Error('Availability assertion was rejected…')` + `code: AVAILABILITY_ASSERTION_REJECTED`, `statusCode: 409` | branch **does NOT match** (no `inventory` substring) → ERROR log → **error rethrown** (propagates to caller) |

**Finding:** live behavior matches static prediction (F-27): the message-substring detector **misses the deterministic authority error** — confirming BR-5-020's rule (code-contract detection, never substrings) is required. E-6 has no failure branch — **BR-5-020 stands regardless**.

**E-6 status: RECORDED (observation complete).** Informative.

---

## CONTRADICTION C-P5-01 — T5-01 halted (domain vs implementation) `[STOPPED — awaiting authority decision]`

**Discovered:** during T5-01 (snapshot contract conformance pins), Stage 6.

**Domain (ratified, 4 statements):**
- **BR-5-005** (`02:605-609`): condition "any source **or the restriction outcome** is unresolved" ⇒ `physicalAvailable = 0`, `sellableAvailable = 0`, `bookingEligibility = 'UNKNOWN'`, **consumption figures null**, `unresolvedSources` surfaced.
- **INV-P5-04** (`03:1131`): "any unresolved source**/dimension** ⇒ `physicalAvailable=0` … consumption figures `null`".
- **REQ-14.2** (`03:633`): under unresolved inputs, `physicalAvailable = 0` and `sellableAvailable = 0` are the mandated conservative floor.
- **P-3 source extract** (`docs/enterprise/availability-phase1-2-contract-extract.md:78-93`): "When any source is unresolved, **or the restriction outcome is unresolved, or a restriction blocks the stay**" ⇒ `consumption: null`, `overbookingUsed: null`, `physicalAvailable: 0`, `sellableAvailable: 0` — labeled **"VERIFIED FACT — `availability-snapshot.service.ts:80,96-102`"**.

**Implementation (actual):** `availability-snapshot.service.ts:99-101` gates `overbookingUsed`/`consumption`/`physicalAvailable` on `sourceUnresolved` **only** (reservation/GBA/allotment sources). When the **restriction outcome is UNRESOLVED** or the stay is **BLOCKED** with all quantity sources resolved, the response returns **`physicalAvailable` = real value, `consumption` = real value** (sellable/eligibility floors at `:103/:105` do apply).

**Runtime evidence (temporary probe spec, executed then deleted):**
- PROBE-A — restriction `UNRESOLVED`, sources resolved: domain expects `{physicalAvailable: 0, consumption: null}` → **observed `{physicalAvailable: 4, consumption: 0}` — FAIL**.
- PROBE-B — restriction RESOLVED + `closedToSell` blocked: P-3 extract expects `{physicalAvailable: 0, consumption: null, eligibility: 'BLOCKED'}` → **observed `{physicalAvailable: 4, consumption: 0, eligibility: 'BLOCKED'}` — FAIL**.

**Internal FDS tension:** AC-04 (`03:1172`) scopes the full floor to "**any source** unresolved"; §13.6 row 3 (`03:609`) mandates only `sellableAvailable=0` + `UNKNOWN` for "any source **or dimension** `UNRESOLVED`"; AC-05/§13.6 row 2 mandate only `sellableAvailable=0` + `BLOCKED` for blocked. The code matches AC-04 (source case) + §13.6 + AC-05, but contradicts BR-5-005 / INV-P5-04 / REQ-14.2 / the P-3 extract table for the restriction-unresolved (and blocked) cases.

**Effect:** T5-01's required pin "unresolved ⇒ `physicalAvailable=0`, …, consumption `null`" **cannot be made green** without either (A) a source change to `availability-snapshot.service.ts` (no `[modify]` task currently authorizes it — T5-01 is `[verify]`/test-only), or (B) a domain clarification/amendment (forbidden without authority: no reinterpretation, no domain drift). Per prompt §2/§37 this path is **STOPPED — no workaround invented, no pin weakened, no code changed**.

**Downstream affected:** T5-01 (contested pin only — all other pins executable), any future task asserting the contested fields under restriction-unresolved/blocked. **Not affected:** assertion/write path (fail-closed already via `unassertableReason` → `UNRESOLVED_CAPACITY`), eligibility/sellable floors, T5-07/09 evaluator work (happy path = RESOLVED dimensions), T5-03 distinctness (state identity via eligibility/labels, passes today).

**Independent executable tasks:** remainder of P0 (T5-01 uncontested pins, T5-02, T5-03, T5-04, T5-12/15/16/18, T5-19/20/21/26/28, T5-29/34/38, T5-45/47, T5-49/51, T5-54/55, T5-57/58/60/61, T5-62/63, T5-64…68, T5-69…75, T5-77…86).

**Authority decision required:** (A) authorize one-line source fix to comply with BR-5-005 (add restriction-outcome/blocked to the floor gate at `:99-101`), or (B) declare BR-5-005/INV-P5-04/REQ-14.2/P-3-extract wording superseded by AC-04+§13.6+AC-05 (domain clarification), or (C) other direction.

---

## E-7 — `/tax-rates` runtime winner (T5-90) `[GATE: E-7, gates T5-27]`

**Executed:** temp trace script (`%TEMP%/opencode/e7-tax-rates-trace.js`) booting a Nest app with the **two real controllers** in exact `activities.module.ts` relative order (`BanquetRefsController` idx2 before `AvailabilitySalesController` idx3), stubbed `PrismaService` with SQL logging, ALS hotel context, live HTTP GET.

**Observations:**
1. Express router: **two** GET `/tax-rates` layers registered (duplicate confirmed at runtime).
2. **Live request executed** `SELECT t.tax_code, t.tax_description, t.tax_percentage, t.tax_type FROM tax_codes t JOIN hotels h …` → **BanquetRefsController's unique query** (single raw join; the availability-sales variant would log `SELECT country FROM hotels …` first). HTTP **200** `[]`.
3. Registration order (module index 2 < 3) aligns with the observed execution.

**E-7 result: runtime winner = `banquet-refs.controller.ts:45` (BanquetRefsController)** — as predicted by registration order. Owner unchanged now (route untouched; current behavior preserved); **T5-27 gate satisfied** (winner recorded). No aliasing attempted.

**E-7 status: RECORDED.**

---

## E-8 — T5-76 (first executed exit) — Status: OPEN

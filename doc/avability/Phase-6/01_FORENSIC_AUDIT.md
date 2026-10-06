# XYLO Availability — Phase 6 Forensic Audit

**Artifact:** `docs/availability/phase-6/01_FORENSIC_AUDIT.md`
**Phase:** 6 (Reconciliation / Cutover / Legacy Retirement) — **pre-decision, pre-plan**
**Mode:** Read-only forensic audit. No code written, no file deleted, no schema touched, no stage or document set invented, no business rule decided.
**Baseline:** workspace authority = local tree `C:\Users\Pro\Desktop\XYLO`, branch `main`, HEAD `c854f79`.
**Working tree at audit time:** 787 dirty paths (34 staged / 170 untracked / 368 modified / 249 deleted); `packages/db` 75; `docs` 35; `apps/api` 142; `apps/web` 310.
**Closed inputs (not re-audited):** Phase 4 (`docs/availability/phase-4/`, 17 artifacts), Phase 5 (`docs/availability/phase-5/`, 6 artifacts, Stage 6 CLOSED, verdict `READY WITH NON-BLOCKING NOTES` at `06_EXECUTION_EVIDENCE.md:2100`).

**Confidence scale used throughout:** `HIGH` = verified by direct source read in this session; `MEDIUM` = derived from two or more corroborating sources, not executed; `LOW` = single-source or inferred, flagged as needing runtime confirmation.

---

## A. Audit Scope, Method and Evidence Basis

| # | Question this audit answers | Method |
|---|---|---|
| A-1 | What is actually built, wired and running for availability as of HEAD? | Static read of `apps/api/src/modules/{availability,activities,reservations,front-office,group-allotment,rates-inventory,crs-integration,shared}`, `apps/web/features/*`, `packages/db/schema.prisma` |
| A-2 | What did Phase 3/4/5 formally defer *into* Phase 6? | Directive quotations from `docs/enterprise/availability-phase3-*`, `docs/availability/phase-4/*`, `docs/availability/phase-5/*` |
| A-3 | Which legacy artefacts still exist, and are they REMOVE / KEEP / DEFER? | Symbol + route + raw-SQL scans with caller enumeration |
| A-4 | Where do two sources of truth still coexist? | Cross-read of every read path and every write path against the authority binding |
| A-5 | What documentation convention must a Phase 6 artifact obey? | Structural comparison of `docs/availability/phase-4/` and `docs/availability/phase-5/` |
| A-6 | Is Phase 6 ready to be planned? | Synthesis of B–O |

**Explicitly out of scope:** runtime execution, test runs, DB connectivity, gateway/edge configuration, GitHub state, any edit to files outside this document.

**Evidence classes referenced:** `E-1…E-8` = Phase 5 evidence gates (`04_IMPLEMENTATION_PLAN.md:253`); `BLK-P5-01…06`; `DS-01…DS-12`; `P-16…P-21` posture decisions; `BR-5-xxx`; `F-01…F-27`; `L-n` = Phase 4 legacy-dependency register (`phase-4/07_LEGACY_DEPENDENCY_MAP.md`).

---

## B. Phase 6 Definition and Boundary (from authorities, not invented here)

Phase 6 has a **written scope definition** in upstream specifications. It does **not** have a document set, a stage structure, or a task register. The following are quotations of record.

| Ref | Source | Statement |
|---|---|---|
| **B-1** | `docs/enterprise/availability-phase3-implementation-plan.md:427` (§13.4 "Reserved for Phase 6") | "Cutover of a Reservation population to assertion authority, deterministic population migration at scale, legacy counter retirement, dual-source reconciliation closure, and removal of `crs-engine` inventory calls." |
| **B-2** | same file `:442` | "Deferred to Phase 6 \| population cutover, legacy counter retirement, dual-source reconciliation." |
| **B-3** | same file `:644-646` | Legacy `availability`/`inventory` counter retirement → Phase 6; population cutover at scale; dual-source reconciliation closure → Phase 6; "Removing `crs-engine` inventory calls from untouched legacy paths" → Phase 6 |
| **B-4** | `docs/enterprise/availability-phase3-domain-specification.md:445` | "Reconciliation, population cutover, and legacy retirement belong to Phase 6; this specification does not perform them." |
| **B-5** | `docs/enterprise/availability-phase3-business-rules-decision-sheet.md:22` (D-20) | "Legacy counter cutover — RESOLVED — deferred to Phase 6" |
| **B-6** | `docs/availability/phase-4/12_DECISION_STATUS.md:17` (D-6) | A1 is sole sellable-capacity authority — "gates Phase 6 integration & D-7" |
| **B-7** | `docs/availability/phase-4/07_LEGACY_DEPENDENCY_MAP.md:36` (L-10) | `room_inventory` — disposition "KEEP read-only or REMOVE (Phase 6)" |
| **B-8** | `docs/availability/phase-5/04_IMPLEMENTATION_PLAN.md:1410-1425` (§32) | Seven Phase-4 carry-overs Phase 5 "may not assume … and may not execute" — all still open at Phase 6 entry |
| **B-9** | `docs/availability/phase-5/04_IMPLEMENTATION_PLAN.md:1429-1442` (§33) | Ten explicitly deferred items, each with authority; none silently dropped |
| **B-10** | `docs/availability/phase-5/06_EXECUTION_EVIDENCE.md:2100-2118` (§34) | Seven non-blocking notes carried out of Phase 5 |

**Boundary fence (observed, not decided):**

*Inside the written Phase 6 scope* — population cutover, legacy counter retirement, dual-source reconciliation closure, `crs-engine` inventory-call removal, L-10 `room_inventory` disposition, carry-over dispositions from B-8, deferred dispositions from B-9 that are tagged Phase 6/11.

*Explicitly not Phase 6* — schema authoring (Deviation C / G-1 holds it), deletion of the legacy `availability` table (P-16 → Phase 11), the 10 items of B-9 that name P-19/P-20/P-21/DS-06/DS-07/DS-08/F-14, and ADR-072 reservation semantics.

**Finding B-AUD-01 — Phase 6 has a scope but no plan.**
`Path:` `docs/availability/phase-6/` (created by this audit, previously absent) · `Symbol:` n/a · `Evidence:` `docs/availability` contained only `phase-4`, `phase-5` before this run · `Status:` OPEN · `Confidence:` HIGH · `Relevance:` Phase 6 must be planned from B-1…B-10, not from a Phase 5 template.

---

## C. Phase 6 Documentation Structure & Artifact Convention

This section is required by the audit brief. It describes **structure and naming conventions that already exist**, and the **differences that follow objectively** from Phase 6 not existing yet. It does **not** name, count, or stage any Phase 6 document beyond this report.

### C.1 Phase 4 structure (reference A)

| Property | Value |
|---|---|
| Location | `docs/availability/phase-4/` |
| Count | 17 artifacts |
| Names | `01_FORENSIC_AUDIT.md`, `02_CURRENT_ARCHITECTURE.md`, `03_DATA_MODEL_AUDIT.md`, `04_AVAILABILITY_INTEGRATION_MATRIX.md`, `05_BUSINESS_RULES_CURRENT_STATE.md`, `06_CONCURRENCY_AND_TRANSACTION_AUDIT.md`, `07_LEGACY_DEPENDENCY_MAP.md`, `08_OPEN_DECISIONS.md`, `09_PHASE_4_AUDIT_SUMMARY.md`, `10_DECISION_RESOLUTION.md`, `11_TARGET_BUSINESS_RULES.md`, `12_DECISION_STATUS.md`, `13_FINAL_DOMAIN_SPECIFICATION.md`, `14_IMPLEMENTATION_PLAN.md`, `15_READINESS_REVIEW.md`, `16_RR_DECISIONS.md`, `17_PHASE4_READINESS_REVIEW.md` |
| Naming | `NN_UPPER_SNAKE_CASE.md`, 2-digit zero-padded, strictly sequential, no gaps |
| H1 convention | Early files: `# Phase 4 — <Title>`; later files: `# XYLO Availability Phase 4 — <Title>` (convention converged mid-phase) |
| Internal structure | Numbered `## 1.`…`## n.` sections with `### n.m` subsections; decision tables keyed `D-n`, `L-n`, `NB-n` |

### C.2 Phase 5 structure (reference B)

| Property | Value |
|---|---|
| Location | `docs/availability/phase-5/` |
| Count | 6 artifacts |
| Names | `01_FORENSIC_AUDIT.md`, `02_BUSINESS_RULES_DECISIONS.md`, `03_FINAL_DOMAIN_SPECIFICATION.md`, `04_IMPLEMENTATION_PLAN.md`, `05_READINESS_REVIEW.md`, `06_EXECUTION_EVIDENCE.md` |
| Naming | identical `NN_UPPER_SNAKE_CASE.md` convention |
| H1 convention | `# XYLO Availability Phase 5 — <Title>` (fully converged), with stage annotation in the H1 of `04`, `05`, `06` |
| Numbering semantics | **1:1 with execution stages** — 01 = Stage 1 forensic, 02 = Stage 2 decisions, 03 = Stage 3 spec, 04 = Stage 4 plan, 05 = Stage 5 readiness, 06 = Stage 6 evidence |
| Internal structure | `## 1.`…`## 37.` in `04`; evidence document uses one `## T5-nn` heading per task plus `## E-n` gate headings and a terminal `## §34 Final Report` |

### C.3 The inherited convention (what a Phase 6 artifact must obey)

1. **Folder** — every Phase 6 markdown artifact lives under `docs/availability/phase-6/`. Nothing Phase 6 is written into `phase-4/`, `phase-5/`, `docs/enterprise/`, `docs/audit/`, or the repo root.
2. **Filename** — `NN_UPPER_SNAKE_CASE.md`, 2-digit zero-padded, allocated sequentially with no gaps and no retroactive renumbering.
3. **H1** — `# XYLO Availability Phase 6 — <Title>`; stage annotation added in the H1 only if Phase 6 is in fact structured in stages (undetermined — see C.4).
4. **Sections** — integer `## n.` with `### n.m`; stable keyed registers (decisions, dependencies, carry-overs, contradictions) use table rows with a short key column, matching Phase 4/5 style.
5. **Terminal artifact** — a phase's last artifact is the evidence/verdict document (Phase 4 used `17_…`, Phase 5 used `06_…`). Its filename and number are **not** predictable until Phase 6's document set is decided.
6. **Citation style** — evidence is cited as `path:line` and gate IDs; claims carry status + confidence.

### C.4 Objectively required differences for Phase 6 (no invention)

| # | Difference | Objective reason |
|---|---|---|
| D-1 | The folder `docs/availability/phase-6/` must be created before any artifact is written | It did not exist at audit start; `docs/availability/` held only `phase-4` and `phase-5` |
| D-2 | Numbering restarts at `01` in the new folder | Both prior phases restart at `01` per folder |
| D-3 | This audit is filed as `01_FORENSIC_AUDIT.md` | Phase 4 `01_FORENSIC_AUDIT.md` and Phase 5 `01_FORENSIC_AUDIT.md` both open the phase with exactly that name — the position and filename are inherited, not chosen |
| D-4 | The **rest** of the Phase 6 document list is undetermined | Phase 4 and Phase 5 use *different* counts (17 vs 6) and *different* numbering semantics (thematic vs stage-bound). No rule in either precedent yields a unique Phase 6 list; asserting one would be invention |
| D-5 | Stage structure, if any, is undetermined | Phase 5 is explicitly stage-bound; Phase 4 is not. Phase 6's own structure has not been decided anywhere in the corpus |
| D-6 | Phase 6 may not "inherit" Phase 5's task/ID scheme (`T5-nn`, `BLK-P5-nn`, `E-n`, `S3R-nnn`) | Those IDs are Phase 5-scoped (`04:1207` §29, `04:1260` §31). Reusing them would corrupt cross-phase traceability |
| D-7 | Phase 6 must reconcile against **three** document trees, not one | `docs/availability/phase-{4,5}/`, `docs/enterprise/availability-phase3-*.md`, and the domain-external `docs/audit/Reservations/` + root `implementation_plan.md` (Reservations Phase 9) — see C.5 |
| D-8 | The Phase 5 terminal artifact is an *input*, not a template | `phase-5/06_EXECUTION_EVIDENCE.md` closes Phase 5; Phase 6 opens from its §34 non-blocking notes and §32/§33 defer tables |

### C.5 Documentation drift risks observed (for Phase 6 to resolve, not resolved here)

| Ref | Observation | Path | Status | Confidence | Phase 6 relevance |
|---|---|---|---|---|---|
| C-5.1 | Root-level `implementation_plan.md` exists for a *different* program (Reservations Phase 9, frontend-only) with no phase-folder affiliation | `implementation_plan.md` | OPEN | HIGH | Risk of being mistaken for an availability plan; Phase 6 must state which trees are authoritative |
| C-5.2 | `docs/RESERVATIONS_FORENSIC_AUDIT.md:445,665` catalogues `POST /rates/engine/modify` as ACTIVE — corroborates the live legacy route but sits in a third tree | `docs/RESERVATIONS_FORENSIC_AUDIT.md` | OPEN | HIGH | Consumer/route inventories must be de-duplicated across trees |
| C-5.3 | Phase 3-era audit lines are stale versus current code: `availability-phase3-reservation-lifecycle-forensic-audit.md:758` says `holdInventory/confirmHold/releaseHold` are "no-op stubs … never injected"; the adapter now implements holds via assertions | `docs/enterprise/availability-phase3-reservation-lifecycle-forensic-audit.md:756-761` vs `apps/api/src/modules/reservations/infrastructure/adapters/prisma-inventory-reservation.adapter.ts:37-70` | OPEN | HIGH | Phase 6 must not consume the Phase 3 legacy register without re-verification |
| C-5.4 | Phase 5 evidence cites stale line anchors for the matrix defect (`availability-sales.controller.ts:296-311`) vs current (`:305-307`, `:333`) | `phase-5/06_EXECUTION_EVIDENCE.md:681` | OPEN | HIGH | Anchor re-check duty (F-23 pattern) carries into Phase 6 |
| C-5.5 | No `phase-6` reference exists in any availability artifact; the only "Phase 6" strings in `docs/` belong to unrelated programs (Reservations `05_FRONTEND_ARCHITECTURE.md:1545`, inventory `ENTERPRISE-ARCHITECTURE-VERIFICATION.md:476`, `design/gba-domain-spec.md:1676`) | `docs/**` | OPEN | HIGH | Confirms the availability Phase 6 doc set is genuinely undetermined |

---

## D. As-Built Authority Topology

| Ref | Symbol / binding | Path | Evidence | Status | Confidence | Phase 6 relevance |
|---|---|---|---|---|---|---|
| D-1 | `RESERVATION_AVAILABILITY_PORT → AvailabilityAssertionService` | `apps/api/src/modules/availability/availability.module.ts:45` | `{ provide: RESERVATION_AVAILABILITY_PORT, useExisting: AvailabilityAssertionService }` | LIVE | HIGH | This is the single write authority. Phase 6 cutover means *every* remaining reservation mutation reaches it |
| D-2 | `RESTRICTION_EVALUATOR → PrismaRestrictionAdapter` (binding-swap, T5-09) | `availability.module.ts:51`; rollback stub retained `:50` | `UnresolvedRestrictionAdapter` still registered as provider `:50` | LIVE | HIGH | Rollback is one line; Phase 6 must not delete the stub |
| D-3 | `RESTRICTION_WRITE_INVALIDATOR` = documented no-op | `availability.module.ts:22-30,44` | `liveReadInvalidator.invalidate = () => undefined` | LIVE (intentional) | HIGH | No availability cache exists (F-14); a future cache is BR-5-039, deferred |
| D-4 | Authority read surface | `availability/api/controllers/availability.controller.ts:9,13,21` | `@Controller('properties/:propertyId/availability')` + `@Get('snapshot')` + `@Get('reconciliation')` — **two routes only** | LIVE | HIGH | Fixed-surface gate `t528-fixed-surface-gate.spec` pins 22 routes |
| D-5 | Authority computation | `availability/domain/policies/snapshot-calculator.ts` + `application/services/availability-snapshot.service.ts` | reads rooms A1 / reservation consumption / GBA / allotment / overbooking | LIVE | HIGH | Sole seller of `sellableAvailable` |
| D-6 | Restriction write path | `application/services/restriction-write.service.ts` + `infrastructure/repositories/restriction-write.repository.ts` | Prisma upserts over 6 stores + `zero_sell_limits.deleteMany`, hotel-scoped | LIVE | HIGH | Consumers: `activities/availability-sales.controller.ts:650` (`POST availability/bulk-update`), `:708` (`POST availability/interval-update`) |
| D-7 | Assert/replace/release transaction engine | `application/services/availability-assertion.service.ts` | bound at `availability.module.ts:45` | LIVE | HIGH | The only sanctioned capacity mutator |
| D-8 | Diagnostics | `infrastructure/reconciliation/availability-reconciliation.service.ts` | sole `prisma.availability.findMany` read; `automaticRepair:false`; pinned `t557-reconciliation-readonly.spec.ts` = exactly `['availability.findMany']` | LIVE, read-only | HIGH | Phase 6 reconciliation *tooling* seed — but diagnostic only today |
| D-9 | Authority write consumers (in-repo) | `reservations/infrastructure/repositories/reservation.repository.ts:559,622,635,695,798,846,962,1020,1226`; `front-office/.../check-out.handler.ts:194,199,215,220`; `front-office/.../upgrade-room.handler.ts:192,198`; `reservations/infrastructure/adapters/prisma-inventory-reservation.adapter.ts:37-70` | all call `RESERVATION_AVAILABILITY_PORT` | LIVE | HIGH | These are the paths already cutover |

**Net:** the authority exists, is bound, is consumed by Reservations + Front Office, and is read by one web hook and one API consult. Everything else still reads or writes legacy.

---

## E. Read-Surface Inventory (availability truth)

### E.1 Authority reads

| Ref | Endpoint / consumer | Path | Status | Confidence | Phase 6 relevance |
|---|---|---|---|---|---|
| E-1 | `GET /api/v1/properties/:propertyId/availability/snapshot` | `availability.controller.ts:13-19` | LIVE | HIGH | Sole canonical read |
| E-2 | `GET /api/v1/properties/:propertyId/availability/reconciliation` | `availability.controller.ts:21-28` | LIVE | HIGH | Flag-less, on-demand legacy-vs-authority comparison |
| E-3 | Web `useAvailabilitySnapshot` → `availabilitySnapshotPath` | `apps/web/features/availability/hooks/use-availability-snapshot.ts:57` | LIVE; consumed only by `features/reservations/workspace/pages/AvailabilityPage.tsx:788` | HIGH | Single UI consumer of canonical snapshot |
| E-4 | Quote authority consult | `rates-inventory/crs-engine.service.ts:246,446-484` (`consultAuthorityEligibility`) | LIVE; `available`/`eligibility`/`signalCodes` from authority `:282-284` | HIGH | Quote is authority-aware; restriction display is not (see H-3) |
| E-5 | Pickup two-layer consult (flag `gba.pickup.twoLayerConsult`) | `group-allotment/infrastructure/adapters/pickup-availability.snapshot-consult.ts:31,140` | LIVE but **flag OFF** | HIGH | Reads `availability_assertion_balances` |
| E-6 | Matrix authority branch (flag `gba.a3.authoritative`) | `activities/availability-sales.controller.ts:301,474` | LIVE, **flag ON** in `.env:68` | HIGH | The only flag currently ON |

### E.2 Legacy reads still live

| Ref | Symbol | Path | Store read | Status | Confidence | Disposition candidate | Phase 6 relevance |
|---|---|---|---|---|---|---|---|
| E-7 | `getAvailabilityMatrix` legacy branch | `activities/availability-sales.controller.ts:340,354,368,382,395` (`authoritative ? [] : …`) | `reservations`+`rooms`, `group_block_daily_allocations`, `allotment_daily_quotas`, `out_of_order`/`out_of_service` | LIVE (flag-dependent) | HIGH | DEFER — flag-OFF rollback path by design (P-17) | Rollback depends on it |
| E-8 | `evaluateRestrictions` | `rates-inventory/crs-engine.service.ts:133-197` | raw `rate_restrictions` `:146-150` | LIVE, unconditional | HIGH | DEFER → Phase 6 dual-source closure | **Reads a store the authority flags as unproven** (H-3) |
| E-9 | `checkAvailability` | `crs-engine.service.ts:86-93` → `inventory.domain-service.ts:56` | raw `availability` `:74` | LIVE; sole caller = `modifyReservation` `:364` | HIGH | REMOVE (Phase 6, per B-1/B-3) | Part of "removal of `crs-engine` inventory calls" |
| E-10 | `getRestrictionRows` | `activities/availability-sales.controller.ts:717` + subselects | reads authority six stores + rooms enumeration (T5-39) | LIVE, authority-backed | HIGH | KEEP | Already cut over |
| E-11 | `room_inventory` reads | `activities/availability-sales.controller.ts:86` (LEFT JOIN), `:276` | `room_inventory` | LIVE, **no writer anywhere** | HIGH | KEEP-read-only or REMOVE (L-10, B-7) | Explicitly a Phase 6 decision (B-7) |
| E-12 | FO integration restriction consult | `crs-integration/crs-front-desk-integration.service.ts:89` | `crs.evaluateRestrictions` → `rate_restrictions` | LIVE | HIGH | DEFER | Second consumer of the unproven store |
| E-13 | Unresolved-restriction fallback | `availability/infrastructure/adapters/unresolved-restriction.adapter.ts:21,36,48,71-77` | emits `rate_restrictions` / `channel_restrictions` as *unresolved* markers | LIVE (rollback adapter) | HIGH | KEEP (rollback artifact) | Documents the very gap H-3 describes |

---

## F. Write-Surface Inventory (capacity mutation)

### F.1 Authority writes

| Ref | Symbol | Path | Status | Confidence |
|---|---|---|---|---|
| F-1 | `assertReservationCreate` / `replaceReservationAssertion` / `applyStatusAvailability` | `reservation.repository.ts:559,622,635,695-705,798-802,962,989-1025` | LIVE | HIGH |
| F-2 | Check-out / upgrade-room assertion writes | `check-out.handler.ts:194,199,215,220`; `upgrade-room.handler.ts:192,198` | LIVE | HIGH |
| F-3 | `InventoryReservationPort.holdInventory` | `prisma-inventory-reservation.adapter.ts:37-70` (asserts `HOLD_SOURCE='RESERVATION_HOLD'`) | LIVE | HIGH |
| F-4 | Restriction writes (6 stores) | `restriction-write.repository.ts` via `availability-sales.controller.ts:650,708` | LIVE | HIGH |

### F.2 Legacy writes still live

| Ref | Symbol | Path | Store written | Reachable via | Status | Confidence | Disposition | Phase 6 relevance |
|---|---|---|---|---|---|---|---|---|
| **F-5** | `CrsEngineService.modifyReservation` | `crs-engine.service.ts:290-444` | **legacy `availability`** via `inventoryDomain.reserve/release` `:391-402`; **`reservations` row** at `:411`; `reservation_changes` `:424-430` | `POST /api/v1/rates/engine/modify` → `rates-inventory.controller.ts:108,123` | **LIVE — external-facing** | HIGH | REMOVE once E-5 known (B-1, B-3) | **Highest-risk Phase 6 item** — see H-1 |
| F-6 | `InventoryDomainService.reserve/release` (sole `INSERT INTO availability`) | `inventory.domain-service.ts:160-188`, raw `:165,182,221,228,248,257,276` | legacy `availability` | F-5 only | LIVE, E-5-gated | HIGH | REMOVE | Confirmed by `t525-engine-release-retirement.spec.ts` allowlist |
| F-7 | Dead legacy methods with **zero callers** | `inventory.domain-service.ts:190 reserveRooms, 198 releaseRooms, 209 blockAvailability, 238 consumePickup, 267 releaseUnsold` | legacy `availability` | none | DEAD | HIGH | REMOVE | Independently corroborated by Phase 3 audit L-5 (`phase3-reservation-lifecycle-forensic-audit.md:756`) |
| F-8 | Dead legacy `crs-engine.checkAvailability` | `crs-engine.service.ts:86-93` | read only | only internal `:364` | DEAD outside F-5 | HIGH | REMOVE | Included in B-1 wording |
| F-9 | Dual-ledger pickup cascade | `reservation-pickup-cascade.service.ts:106` (canonical `group_pickups`) + `:113` (legacy `allotment_pickups`) | both, gated by `gba.pickup.canonicalWrite` (`:102`) | `events.consumer.ts:34,151,172`, gated by `gba.consumers.cascade` | LIVE but **both flags OFF** | HIGH | DEFER — flagged sequencing | **Neither ledger is being cascaded right now** (J-3) |
| F-10 | Channel availability writer | `channels/channels.service.ts` → `channel_availability`, `channel_availability_log` | channel tables | channels module | LIVE | HIGH | OUT OF SCOPE (separate bounded context) | Must be fenced, not touched |
| F-11 | A3 operation log writer | `availability-sales.controller.ts:693-700` → `availability_matrix_logs` | log table | `POST availability/logs` | LIVE | HIGH | KEEP (never authority — `t561-logs-never-authority.spec`) | Non-authoritative by design |

**Retired and confirmed absent** (404 by construction, pinned): `@Get('engine/availability')`, `@Get('engine/restrictions')`, `@Get('rates/availability')`, `@Post('engine/book')`, `@Post('engine/release')` — `t522/t523/t524/t525` specs; `rates-inventory.controller.ts` now ends at `:108 engine/modify`.

**Retained legacy routes** (`rates-inventory.controller.ts`): `:34 engine/quote`, `:46 engine/rate`, `:108 engine/modify`, plus `plans`, `calendar`, `policy/*`.

---

## G. Legacy Dependency Inventory — REMOVE / KEEP / DEFER

Every row: path + symbol + evidence + status + confidence + Phase 6 relevance.

| Ref | Artefact | Path / symbol | Evidence | Status | Confidence | Classification | Phase 6 relevance |
|---|---|---|---|---|---|---|---|
| G-1 | legacy `availability` table | `packages/db/schema.prisma:433-453` | only touched by `inventory.domain-service.ts` (10 raw statements); pinned read-only by P-16/T5-41 | Present; 1 live writer (F-6) | HIGH | **KEEP** (read-only) → deletion is **Phase 11** | B-9 explicitly defers deletion |
| G-2 | `InventoryDomainService` | `inventory.domain-service.ts:56-286` | callers enumerated at §F.2 | LIVE behind E-5 | HIGH | **REMOVE** | Direct B-1/B-3 item |
| G-3 | `crs.modifyReservation` + `POST rates/engine/modify` | `crs-engine.service.ts:290`; `rates-inventory.controller.ts:108` | zero in-repo callers (web uses only `:91 quote`, `/reservations` create) | LIVE, external unknown | HIGH | **DEFER (E-5)** → then REMOVE | B-1 "removal of `crs-engine` inventory calls" |
| G-4 | `evaluateRestrictions` | `crs-engine.service.ts:133` | 3 live callers (`:245,:375`, FO integration `:89`, unresolved adapter `:25`) | LIVE | HIGH | **DEFER** | Dual-source closure (B-1) |
| G-5 | `room_inventory` model | `schema.prisma:10413`; readers `availability-sales.controller.ts:86,276` | no writer anywhere | Present, read-only | HIGH | **KEEP read-only** *or* **REMOVE** — undetermined | Explicit B-7 Phase 6 decision |
| G-6 | `channel_restrictions` model | `schema.prisma:2056` | "no writer; excluded until a writer exists" `prisma-restriction.adapter.ts:23,116` | Present, unread by authority | HIGH | **KEEP** (excluded) | BR-5-046 exclusion stands until a writer exists |
| G-7 | `restrictions` (CUTOFF flavour) | `schema.prisma:10087` | excluded E-4/BR-5-024 `prisma-restriction.adapter.ts:24` | Present, excluded | HIGH | **KEEP** (excluded) | Meaning recorded by E-4 |
| G-8 | `rate_restrictions` | `schema.prisma:8925`; reader `crs-engine.service.ts:147` | **writer UNKNOWN** — no in-repo writer found; authority marks `unproven` (`prisma-restriction.adapter.ts:114,153`) | Present, unresolved | HIGH | **DEFER** — E-1 gate | Blocks honest restriction cutover |
| G-9 | `channel_availability` / `channel_availability_log` | `schema.prisma:1814` | writer = `channels/channels.service.ts` | LIVE | HIGH | **KEEP** (out of availability scope) | Must be fenced |
| G-10 | `rate_restrictions`-flavoured `allotment_pickup` legacy ledger | `schema.prisma:137`; table `allotment_pickups` | read `allotment.controller.ts:338,346,379`; write `reservation-pickup-cascade.service.ts:113` (flag-gated) | Present; writer flag-OFF | HIGH | **KEEP → REMOVE after `gba.pickup.canonicalWrite` ON** | B-9 sequencing (`canonicalRead`→soak→`canonicalWrite`) |
| G-11 | Orphaned adapters `prisma-front-office-handoff.adapter.ts`, `prisma-guest-profile.adapter.ts` | `apps/api/src/modules/reservations/infrastructure/adapters/` | zero external references; Phase 5 §34 note 4 | Present, untracked | HIGH | **DEFER** — flagged for Phase-11 disposal | Phase 5 note 4 explicitly reserves them |
| G-12 | `PrismaInventoryReservationAdapter` | same dir | sole F-03/F-27 pin via `reservation-hold-mechanism.postgres.spec.ts` | LIVE | HIGH | **KEEP** | Do not delete |
| G-13 | `UnresolvedRestrictionAdapter` | `availability.module.ts:50` | retained provider for T5-09 binding revert | LIVE | HIGH | **KEEP** | Rollback mechanism (§28) |
| G-14 | Legacy client allowlist | `apps/web/features/availability/__tests__/ws-n-scan-gates.test.ts:118-122` (`CLIENT_MATH_ALLOWED`) and the API-side counterpart `apps/api/src/modules/availability/infrastructure/__tests__/ws-n-scan-gates.spec.ts` (families T5-77…T5-86) | allowlists = `group-allotment/views/`, `app/(dashboard)/inventory/`, `app/(dashboard)/lost-found/` | LIVE | HIGH | **REMOVE entries as each site is retired** | These allowlists are the primary retirement-candidate register |
| G-15 | Retired-route delay allowlist | `apps/web/app/(dashboard)/rates-inventory/page.tsx:140` (`/rates/engine/rate`), `apps/web/features/reservations/hooks/use-crs-book.ts:91` (`/rates/engine/quote`), `apps/web/features/front-office/api/front-office.api.ts:193` (`/rates/engine/quote`) | T5-80 "delay retirement" disposition | LIVE | HIGH | **DEFER** until engine routes retire | Directly gates F-5 removal |
| G-16 | Dead `this.crs` / `inventoryDomain` DI in repository | `reservation.repository.ts:5,15,83,89` | pinned as intentional by `ws-n-scan-gates.spec.ts:187` | LIVE (dead DI) | HIGH | **REMOVE** after F-5 retires | Hygiene, not behaviour |

---

## H. Divergence / Dual-Source Register (where two truths coexist)

| Ref | Divergence | Exact paths | Mechanism | Status | Confidence | Phase 6 relevance |
|---|---|---|---|---|---|---|
| **H-1** | **Reservation row vs assertion state** | `crs-engine.service.ts:411` (`tx.reservations.update`) executes **without** any `RESERVATION_AVAILABILITY_PORT` call, inside `modifyReservation` (`:388-432`) | External `POST /rates/engine/modify` changes arrival/departure/room_type on `reservations` and moves **legacy counters only**; `reservation_availability_state` / `availability_assertion_balances` are never replaced | LIVE | HIGH | **The core reconciliation problem.** Every external modify silently desynchronises authority from the reservation row |
| **H-2** | **Legacy counter vs assertion balance** | `inventory.domain-service.ts:165,182,221,228,248,257,276` write `availability`; authority writes `availability_assertion_balances` | Two counters for the same capacity, no reconciler comparing them | LIVE | HIGH | `AvailabilityReconciliationService` compares **legacy projection vs snapshot**, not counter-vs-balance — see I-2 |
| **H-3** | **Two restriction interpretations** | (a) `crs-engine.service.ts:146-196` treats `rate_restrictions` as a hard blocker for quotes; (b) `prisma-restriction.adapter.ts:114,153` treats it as `unproven` → contributes `UNRESOLVED`, never `BLOCKED` | Same store, opposite epistemic status, both live simultaneously | LIVE | HIGH | Quote can be `blocked=true` while the authority reports `UNRESOLVED`. Phase 6 "dual-source reconciliation closure" must resolve this |
| **H-4** | **`getQuote` composite truth** | `crs-engine.service.ts:244-288`: `blocked` from legacy restrictions `:250`, `available`/`eligibility` from authority `:282` | A single response field pair sourced from two authorities | LIVE | HIGH | User-visible inconsistency class |
| **H-5** | **Dual pickup ledgers** | canonical `group_pickups` vs legacy `allotment_pickups`; writer `reservation-pickup-cascade.service.ts:106,113` | Both written while `gba.pickup.canonicalWrite` is OFF | LIVE, **currently neither written** (cascade flag OFF) | HIGH | B-9 sequencing item |
| **H-6** | **Matrix dual branch** | `availability-sales.controller.ts:301` flag selects authority `:474` vs legacy `:340…:407` | Same endpoint, two computations, distinguished only by a response `label` | LIVE, flag ON | HIGH | Rollback = flag OFF (P-17) |
| **H-7** | **Legacy `checkAvailability` vs authority eligibility** | `modifyReservation` gates on `checkAvailability` (`:364-369`) — legacy counters — while `getQuote` gates on authority (`:282`) | Same guest flow, different gate | LIVE | HIGH | Booking through modify can succeed where quote says blocked, or vice-versa |

---

## I. Reconciliation Machinery As-Built

| Ref | Capability | Path | What it compares | Repair | Status | Confidence | Phase 6 relevance |
|---|---|---|---|---|---|---|---|
| I-1 | Availability projection reconciliation | `apps/api/src/modules/availability/infrastructure/reconciliation/availability-reconciliation.service.ts` | authority snapshot vs legacy `availability` projection, row-by-row | `automaticRepair:false`, diagnostic only; sole `availability.findMany` read | LIVE, exposed at `GET properties/:propertyId/availability/reconciliation` (`availability.controller.ts:21-28`), no cron, no flag | HIGH | **The only availability reconciler.** No scheduled run, no repair, no persistence of findings beyond the response |
| I-2 | Counter-vs-balance reconciliation | *none* | legacy `availability.available` vs `availability_assertion_balances.asserted_quantity` | — | **ABSENT** | HIGH | Directly required by B-1 "dual-source reconciliation closure" |
| I-3 | Population census / cutover tooling | *none* — `reservation_availability_state.population ∈ {LEGACY, ASSERTION_MANAGED}` exists (`schema.prisma` CHECK, Phase 3) but nothing assigns `LEGACY` and no migration tool exists | — | — | **ABSENT** | HIGH | B-1 "deterministic population migration at scale" is unserviced |
| I-4 | GBA reconciliation detectors | `group-allotment/infrastructure/reconciliation/gba-reconciliation.service.ts:122` | 6 detectors incl. `LEGACY_DRIFT` (legacy `allotment_pickups` vs `group_pickups`) `:350` vs `:357` | report-only, every statement a `SELECT` | LIVE but **flag OFF** (`gba.reconciliation.enabled`); `@Cron(EVERY_HOUR)` `:131`; on-demand `runReconciliation(hotelId)` ignores flag | HIGH | The only *scheduled* reconciler in the codebase, and it is about GBA, not availability |
| I-5 | Wash scheduler | `group-allotment/infrastructure/schedulers/wash-scheduler.service.ts:31` `@Cron(EVERY_HOUR)` | n/a | — | LIVE but **flag OFF**; blocked on Deviations A+B | HIGH | Phase 4 carry-over BLK-1/BLK-2 (B-8) |
| I-6 | Outbox event for assertion lifecycle | *none* | Phase 3 audit M-10: "New outbox event(s) for assertion lifecycle (reconciliation/Phase 6)" = MISSING | — | ABSENT | HIGH | Phase 6 will need an event to reconcile asynchronously |
| I-7 | Scheduled/repair job for availability | *none* — no `@Cron` under `modules/availability` | — | — | ABSENT | HIGH | Confirms I-1 is on-demand only |

---

## J. Feature-Flag and Configuration State

| Ref | Flag | `.env:68` | `.env.example:100-106` | Code default | Effect when OFF | Status | Confidence | Phase 6 relevance |
|---|---|---|---|---|---|---|---|---|
| J-1 | `gba.a3.authoritative` | **true** | false | `'false'` | matrix serves labelled legacy view (P-17) | **ON — the only ON flag** | HIGH | Rollback = remove/false the line (§34 note 7) |
| J-2 | `gba.consumers.cascade` | **absent → false** | false, annotated "**DEPLOY INVARIANT: MUST be ON at deploy** (Phase 4 §6, FDS §28.3)" | false | pickup cascade skipped | **OFF — deploy invariant unsatisfied** | HIGH | **Finding:** documented deploy action was never executed (plan-sanctioned: `04:1421` "documented, not executed in code"; T5-54 documents only) |
| J-3 | `gba.wash.schedulerEnabled` | absent → false | false, "BLOCKED on Deviations A+B" | false | no wash | OFF, correctly | HIGH | Blocked by B-8 carry-overs |
| J-4 | `gba.reconciliation.enabled` | absent → false | false, "OFF until the soak sequence (FDS §28.3)" | false | hourly GBA recon dormant | OFF | HIGH | Phase 6 must decide activation order |
| J-5 | `gba.pickup.twoLayerConsult` | absent → false | false | false | quota-only guards (documented A-12 risk) | OFF | HIGH | Pickup ignores authority today |
| J-6 | `gba.pickup.canonicalRead` | absent → false | false, "sequencing: canonicalRead → soak → canonicalWrite" | false | reads legacy `allotment_pickups` (`allotment.controller.ts:346,379`) | OFF | HIGH | First step of B-9 sequence |
| J-7 | `gba.pickup.canonicalWrite` | absent → false | false, "after canonicalRead + soak" | false | dual-ledger writes (`reservation-pickup-cascade.service.ts:102`) | OFF | HIGH | Last legacy pickup writer frozen only when ON |

**Finding J-AUD-01 — flag posture is internally inconsistent with the documented rollout.**
`Path:` `.env:68` + `.env.example:101` · `Symbol:` `FEATURE_GBA_A3_AUTHORITATIVE=true` vs `FEATURE_GBA_CONSUMERS_CASCADE=false` · `Evidence:` A3 is ON (T5-56 discharged) while its documented deploy invariant (cascade ON) and the entire downstream sequence (J-4…J-7) remain OFF · `Status:` OPEN · `Confidence:` HIGH · `Relevance:` Phase 6 sequencing must start from an as-is flag census, not from the documented end-state.

---

## K. Consumer Inventory

### K.1 In-repo API consumers of availability

| Ref | Consumer | Path | Uses | Status | Confidence |
|---|---|---|---|---|---|
| K-1 | Reservations repository | `reservation.repository.ts` | authority port | CUTOVER | HIGH |
| K-2 | FO check-out / upgrade-room | `check-out.handler.ts`, `upgrade-room.handler.ts` | authority port | CUTOVER | HIGH |
| K-3 | `ReservationAvailabilityPort` holds | `prisma-inventory-reservation.adapter.ts` | authority port | CUTOVER | HIGH |
| K-4 | Activities matrix | `availability-sales.controller.ts:290` | authority (flag ON) + legacy branch | MIXED | HIGH |
| K-5 | CRS quote | `crs-engine.service.ts:199,446` | authority consult + legacy restrictions | MIXED | HIGH |
| K-6 | CRS modify | `crs-engine.service.ts:290` | **legacy only** | NOT CUTOVER | HIGH |
| K-7 | CRS front-desk integration | `crs-integration/crs-front-desk-integration.service.ts:89` | legacy restrictions | NOT CUTOVER | HIGH |
| K-8 | Pickup consult | `group-allotment/.../pickup-availability.snapshot-consult.ts` | authority (flag OFF) | DORMANT | HIGH |
| K-9 | Pickup counter read | `group-allotment/api/controllers/allotment.controller.ts:313,338-379` | canonical (flag OFF → legacy) | DORMANT | HIGH |
| K-10 | Pickup cascade | `shared/events.consumer.ts:34,151,172` | dual ledger (flag OFF) | DORMANT | HIGH |
| K-11 | A3 restriction writes | `availability-sales.controller.ts:650,708` | `RestrictionWriteService` | CUTOVER | HIGH |

### K.2 Frontend

| Ref | Consumer | Path | Endpoint | Status | Confidence |
|---|---|---|---|---|---|
| K-12 | `useAvailabilitySnapshot` | `apps/web/features/availability/hooks/use-availability-snapshot.ts:57` | `GET /properties/:id/availability/snapshot` | LIVE (1 consumer) | HIGH |
| K-13 | `useAvailabilityMatrix` | `apps/web/features/reservations/hooks/use-reservation-queries.ts:76` → `reservation.api.ts:279` | `GET /availability/matrix` | LIVE (2 consumers: `AvailabilityPage.tsx:782`, `AvailableRatesMatrix.tsx:40`) | HIGH |
| K-14 | Restriction rows / logs / bulk / interval | `reservation.api.ts:294,444,452,467,477` | `/availability/*` | LIVE | HIGH |
| K-15 | Quick-book | `use-crs-book.ts:91` (`/rates/engine/quote`), `:135` (`POST /reservations`) | mixed legacy quote + authority create | MIXED | HIGH |
| K-16 | Front-office rate | `apps/web/features/front-office/api/front-office.api.ts:193` (`/rates/engine/quote`) | legacy quote | MIXED | HIGH |
| K-17 | Reconciliation UI | *none* | `GET …/availability/reconciliation` has **no web caller** | ABSENT | HIGH |

### K.3 Other surfaces

| Ref | Surface | Evidence | Status | Confidence |
|---|---|---|---|---|
| K-18 | Admin panel | `rg availability apps/admin` → **0 files** | ABSENT (P-19 greenfield rule, B-9) | HIGH |
| K-19 | Mobile | `rg availability apps/mobile` → **0 files** | ABSENT (M-7 greenfield rule, B-9) | HIGH |
| K-20 | **External `/rates/engine/*` consumers** | Phase 5 **E-5 = UNKNOWN** (`06_EXECUTION_EVIDENCE.md:100`, `:2107`): no gateway container, no nginx, no ops logs in repo | **UNKNOWN — BLOCKING for external retirement** | HIGH (as to its unknown-ness) |

---

## L. Evidence-Coverage Assessment (Phase 5 closure integrity, read-only)

Phase 5's `§34` verdict asserts *"every task closed with recorded evidence"* (`06_EXECUTION_EVIDENCE.md` final line). Measured against the plan's own task register:

| Ref | Measurement | Result | Status | Confidence | Phase 6 relevance |
|---|---|---|---|---|---|
| L-1 | Tasks defined in `04_IMPLEMENTATION_PLAN.md` | **90** (`T5-01`…`T5-90`) | confirmed | HIGH | Task registers must be countable |
| L-2 | `## T5-nn` headings in `06_EXECUTION_EVIDENCE.md` | **70** | confirmed | HIGH | 20 tasks covered by `## E-n`, `## T5-77 … T5-86 (combined)`, or range notation |
| L-3 | Task IDs **never appearing anywhere** in the evidence document | **`T5-11`, `T5-36`** | **GAP** | HIGH | See L-6 |
| L-4 | Task IDs with no dedicated heading, mentioned only as preconditions | `T5-27` (covered under `E-7`, gate recorded `:199-208`), `T5-41` (only "T5-83 re-derives" `:1229`), `T5-59` (explicitly *"not in the §29 P0 list"* `:850`) | PARTIAL | HIGH | See L-7 |
| L-5 | Task IDs absent from §29 execution order (excluding range-notation artifacts) | `T5-11, T5-27, T5-36, T5-41, T5-44, T5-59` | OPEN | MEDIUM | `T5-44` *does* have a heading (`:1981`) |
| L-6 | `T5-11` — "Interim gate honesty test" (`04:415`) and `T5-36` — "GBA/ledger views: server values, no client recomputation" (`04:832`) | **No execution record.** `T5-36`'s target client math still exists: `apps/web/features/group-allotment/views/GroupBookingsListView.tsx:101` (`pickupPct`), `GroupBookingDetailView.tsx:101,109,114` — permitted by `CLIENT_MATH_ALLOWED` (`ws-n-scan-gates.test.ts:119`) | **NOT EVIDENCED** | HIGH | These are open Phase 6 items, not closed ones |
| L-7 | `T5-59` — "Delete-reject-when-availability-state" (BR-5-043, AC-34) | `RESERVATION_HAS_AVAILABILITY_STATE` is **defined but never thrown**: sole occurrence `apps/api/src/common/exceptions/availability-error-codes.ts:9,21`. `reservation.repository.ts:878-913` `delete()` has **no** availability-state check; evidence `:858` itself states delete still surfaces *"the exact raw-error class T5-59 will convert to typed"* | **NOT IMPLEMENTED** | HIGH | Phase 6 must either schedule it or formally descope it |
| L-8 | Exit battery §34 (12 gates) | all recorded PASS `:2088-2092` | CLOSED | HIGH | — |
| L-9 | Non-blocking notes | 7 notes `:2104-2112` | CARRIED | HIGH | Direct Phase 6 inputs |

---

## M. Contradiction Register (documentation vs implementation)

| Ref | Documentation says | Implementation shows | Paths | Status | Confidence | Phase 6 relevance |
|---|---|---|---|---|---|---|
| **M-1** | "every task closed with recorded evidence" (§34 closing line) | `T5-11`, `T5-36` never appear; `T5-59` not implemented; `T5-41` only re-derived indirectly | `06_EXECUTION_EVIDENCE.md` §34 vs `04:415,832,886` | **CONTRADICTION** | HIGH | Phase 6 entry must accept or correct this |
| **M-2** | "legacy `availability` table + `inventory.domain-service` retained **read-only** (P-16/T5-41, never dropped)" (`06:2092`) | `inventory.domain-service.reserve/release` are **live writers** behind `POST /rates/engine/modify` (`crs-engine.service.ts:391-402`, `rates-inventory.controller.ts:108`) | `06:2092` vs `crs-engine.service.ts:391` | **CONTRADICTION** (read-only is true only *within the repository path*; the external route still writes) | HIGH | "Retired" language overstates the actual state |
| **M-3** | Phase 3 audit: `InventoryReservationPort.holdInventory/confirmHold/releaseHold` are "no-op stubs … never injected" (`L-7`) | adapter implements `holdInventory` via assertions, is injected and pinned by a postgres spec | `docs/enterprise/availability-phase3-reservation-lifecycle-forensic-audit.md:758` vs `prisma-inventory-reservation.adapter.ts:37-70` | STALE DOC | HIGH | C-5.3 — upstream registers must be re-verified |
| **M-4** | Phase 5 evidence anchors the matrix defect at `availability-sales.controller.ts:296-311` | anchors are now `:305-307` and `:333` | `06:681` vs `availability-sales.controller.ts:305-337` | STALE DOC | HIGH | C-5.4 |
| **M-5** | `.env.example:101` — cascade "DEPLOY INVARIANT: MUST be ON at deploy" | not present in `.env` at all | `.env.example:101` vs `.env:68` | **CONFIG GAP** (documented as operational, never executed) | HIGH | J-2 |
| **M-6** | P-16/T5-41 "read-only table, no new readers" | `availability-reconciliation.service.ts` performs `availability.findMany` — sanctioned and pinned, but it *is* a reader | `t557-reconciliation-readonly.spec.ts` | RECONCILED (sanctioned exception) | HIGH | Baseline for any new reader |
| **M-7** | `Phase 5 §33` defers "Legacy `availability` table deletion" to Phase 11 | consistent | `04:1437` | CONSISTENT | HIGH | — |
| **M-8** | `06:1692` "legacy-counter writer set from booking/release paths = ∅ in-repo; the only remaining legacy writer path is external `engine/modify`" | verified true | `crs-engine.service.ts:391-402` | **CONSISTENT** | HIGH | Corroborates M-2's nuance |

---

## N. Defects and Risks Found (no fix applied)

### N-1 — **LIVE DEFECT:** `GET /availability/matrix?roomType=…` fails when a room-type filter is supplied

| Field | Value |
|---|---|
| **Path** | `apps/api/src/modules/activities/availability-sales.controller.ts` |
| **Symbol / endpoint** | `getAvailabilityMatrix` (`@Get('availability/matrix')` `:290`), query 2 `:326-337` |
| **Mechanism** | `rtFilterRt` is built at `:306` as ``AND UPPER(rt.room_type) = …`` and interpolated at `:333` into a query whose only aliases are `rc` and `rd` (`FROM rate_codes rc JOIN rate_details rd` `:329-330`). Postgres: *missing FROM-clause entry for table "rt"*. |
| **Trigger condition** | `roomType` query parameter non-empty. This query executes **unconditionally** (`:326`, not gated by `authoritative`), so it breaks in **both** flag states. |
| **Reachable from** | `apps/web/features/reservations/workspace/pages/AvailabilityPage.tsx:785` (`roomType: roomTypeFilter \|\| undefined`, dispatched unconditionally at `:782`) and `apps/web/features/reservations/workspace/components/quick-book/AvailableRatesMatrix.tsx:40` |
| **Reference evidence** | Phase 5 recorded it as a **pre-existing, NOT fixed** observation: `06_EXECUTION_EVIDENCE.md:681` — *"SQL error whenever a `roomType` filter is supplied … soak runs pass `roomType=undefined`"*; recorded for "T5-22/23 controller-surface work", which was a route-retirement task and did not address it. No later record shows a fix. |
| **Secondary** | `rtFilter` (`:305`, alias `r`) is interpolated into query 1 (`FROM rooms r` — valid), query 3 (`:349`, aliases `res`/`rm` — invalid) and query 6 (`:400`, alias `rm` — invalid). Queries 3 and 6 are `authoritative ? [] : …`, so they break **only when the flag is OFF**. |
| **Why tests miss it** | `t38-authoritative-matrix.spec.ts` and `t520-matrix-projection-conformance.spec.ts` mock Prisma; `apps/web/.../t568-property-context.test.ts:62` asserts URL construction only. |
| **Status** | OPEN — unaddressed at Phase 5 exit |
| **Confidence** | HIGH (static); not executed against a live DB in this audit |
| **Phase 6 relevance** | Class **technical defect**, not a business decision. It is on the A3 authority surface that Phase 6 must keep healthy. Also re-opens the E-8/§34 anchor claim in M-4. |

### N-2 — External modify path performs an unaudited cross-domain write

| Field | Value |
|---|---|
| **Path / symbol** | `crs-engine.service.ts:388-432` inside `modifyReservation`; entry `rates-inventory.controller.ts:108,123` |
| **Mechanism** | Direct `tx.reservations.update` `:411`, `reservation_changes` insert `:424`, `dailyElements.upsertMany` `:415`, legacy counter reserve/release `:391-402` — all inside one transaction, with **no** availability-authority call and **no** cross-check against `reservation_availability_state`. |
| **Reference evidence** | Phase 5 deliberately retained it: `06:1642` "Retained: `crs.modifyReservation` (service) + `@Post('engine/modify')` (controller) — external-facing, E-5 UNKNOWN ⇒ timing waits"; FDS `03:834` classifies it "GOVERNED BY §24", not retired. |
| **Status** | OPEN (E-5 blocked) |
| **Confidence** | HIGH |
| **Phase 6 relevance** | This is B-1's "removal of `crs-engine` inventory calls" verbatim, and H-1's generator |

### N-3 — Deploy invariant not satisfied

Cascade flag absent from `.env` while `.env.example:101` declares it a MUST-be-ON deploy invariant. Effect: `events.consumer.ts:151,172` never call the pickup cascade, so neither `group_pickups` nor `allotment_pickups` is updated on reservation cancel/checkout from that path.
**Status:** OPEN (plan-sanctioned as an operational action, `04:1421`) · **Confidence:** HIGH · **Class:** technical/operational decision.

### N-4 — Reconciliation has no persistence, schedule or repair

`AvailabilityReconciliationService` returns a report per request; there is no cron, no finding store, no `automaticRepair`, and no controller-side persistence. GBA's reconciler (`gba-reconciliation.service.ts:131`) *is* scheduled but flag-OFF and out of availability scope.
**Status:** OPEN · **Confidence:** HIGH · **Class:** technical design decision for Phase 6.

### N-5 — Duplicate `GET /tax-rates` route (E-7, recorded, unresolved as a merge)

`availability-sales.controller.ts:155` and `banquet-refs.controller.ts:45`, both `@Controller()`, both under global prefix `api/v1` (`main.ts:21`). Phase 5 executed a runtime trace and recorded the winner (`06:199-208`): `BanquetRefsController` by registration order; "No aliasing attempted"; "backend merge owned by T5-27/T5-90 disposition".
**Status:** OPEN (recorded, non-blocking) · **Confidence:** HIGH · **Class:** technical decision.

### N-6 — Raw SQL with interpolated identifiers/filters on the A3 surface

`getAvailabilityMatrix` builds `rtFilter`/`rtFilterRt`/`rcFilter` by string interpolation (`:305-307`) with quote-escaping only, then passes them to `$queryRawUnsafe`. Also `:377` and `:390` embed `roomType` directly.
**Status:** OPEN · **Confidence:** HIGH (pattern present); injection risk assessed as mitigated by `replace(/'/g,"''")` but identifier interpolation is fragile — this is what produces N-1 · **Class:** technical.

---

## O. Decision Register — Business vs Technical (flagged only, none decided here)

### O.1 Decisions that are genuinely **business / domain** (require a human owner)

| Ref | Decision | Why it is business | Blocking evidence | Confidence |
|---|---|---|---|---|
| O-1 | Is `room_inventory` removed, or kept read-only? | Changes what an operations user sees and what reports mean | B-7 / L-10 | HIGH |
| O-2 | Does `rate_restrictions` remain a hard quote blocker while the authority calls it `UNRESOLVED`? | Determines whether a guest can be told "no" by a source the system itself distrusts | H-3, E-8 | HIGH |
| O-3 | Is external `/rates/engine/modify` still used by any third party, and if so, on what date does it stop? | External contract; E-5 UNKNOWN | `06:100,2107` | HIGH |
| O-4 | When does `gba.consumers.cascade` become ON, and is "on-deploy" still the correct rule? | Operational change-management | `.env.example:101`, `04:1421` | HIGH |
| O-5 | Ordering of `canonicalRead` → soak → `canonicalWrite`, and what "soak" means in duration/evidence terms | Availability/pickup cutover semantics | `.env.example:105-106`, B-9 | HIGH |
| O-6 | Should reservation delete reject with a typed 409 when availability state exists (BR-5-043), or is raw FK rejection acceptable indefinitely? | Product behaviour on a destructive operation | L-7 | HIGH |
| O-7 | Should `pickupPct` client math in GBA views be replaced by server values (T5-36), or is the allowlist permanent? | What operators see on the GBA screens | L-6, G-14 | HIGH |
| O-8 | Disposition of Phase 4 carry-overs BLK-1 (GUARANTEED_BLOCK wash exclusion) and BLK-2 (durable wash/release/attrition store) | Product rules for group blocks | `04:1416-1417` | HIGH |
| O-9 | Availability analytics contract (canonical occupancy formula; real-time vs batch) | Measurement definition | `04:1433` (P-20, DS-06, BR-5-029) | HIGH |
| O-10 | Channel/push outbound availability publication semantics | External distribution policy | `04:1434` (P-19, DS-07, BR-5-030) | HIGH |

### O.2 Decisions that are **technical / engineering**

| Ref | Decision | Options visible in code today | Confidence |
|---|---|---|---|
| O-11 | Fix N-1 (`rtFilterRt` alias) — and whether to parameterise the interpolated filters | correct alias, or parameterised query | HIGH |
| O-12 | Whether `modifyReservation` should call the assertion port or be retired outright once E-5 is known | port call vs route retirement | HIGH |
| O-13 | Reconciliation persistence + cadence for availability (counter-vs-balance) | extend `AvailabilityReconciliationService` vs new scheduled job vs outbox consumer (I-6) | HIGH |
| O-14 | Population cutover mechanism (I-3) — how `LEGACY` → `ASSERTION_MANAGED` is assigned at scale, given D-21 forbids bulk backfill (`04:438`) | first-touch (already `T-07`) vs bounded batches | HIGH |
| O-15 | Whether `evaluateRestrictions` becomes an authority consult or stays a `rate_restrictions` read | consult (like `consultAuthorityEligibility`) vs keep | HIGH |
| O-16 | Merge of the duplicate `/tax-rates` routes (N-5) | merge vs alias vs leave | HIGH |
| O-17 | Whether the Phase 3-era legacy register (`L-n`) is re-baselined or annotated (C-5.3) | annotate vs rewrite | HIGH |
| O-18 | Whether `T5-11`, `T5-36`, `T5-41`, `T5-59` are retro-closed, re-scoped into Phase 6, or descoped | three viable paths | HIGH |

---

## P. Entry Readiness: What Phase 6 Actually Requires, and What It Does Not

### P.1 Work that is **NOT** required (verified absent or already done — do not re-do)

| Ref | Item | Evidence | Confidence |
|---|---|---|---|
| P-1 | Re-auditing or reopening Phase 5 | Phase 5 Stage 6 CLOSED, 12/12 gates PASS, `06:2088-2092` | HIGH |
| P-2 | Building the assertion engine, snapshot, restriction evaluator, restriction write, reconciliation service | All present and bound, §D | HIGH |
| P-3 | Re-pointing Reservations / Front Office / holds to the authority | §D-9 | HIGH |
| P-4 | Retiring `engine/availability`, `engine/restrictions`, `engine/book`, `engine/release`, `rates/availability` | 404 by construction, `t522/523/524/525` | HIGH |
| P-5 | Removing the repository's legacy modify leg (T5-50) | `reservation.repository.ts` authority-only at `:622` | HIGH |
| P-6 | Deleting the four duplicate top-level allotment/group-block pages (T5-43) | `06:1948` | HIGH |
| P-7 | Authoring schema or migrations for Phase 6 | Phase 3 declared "no new schema"; Deviation C; `git status packages/db` = 75 pre-existing paths | HIGH |
| P-8 | Building admin/mobile availability surfaces | P-19/M-7 greenfield rule; K-18/K-19 = 0 files | HIGH |
| P-9 | Any GitHub fetch/pull/restore | Not performed; local tree is authority | HIGH |

### P.2 Entry blockers (must be dispositioned before Phase 6 is planned)

| Ref | Blocker | Class | Evidence | Confidence |
|---|---|---|---|---|
| **X-1** | **E-5 external consumer inventory UNKNOWN** | technical-unknown | `06:100,2107` | HIGH |
| **X-2** | M-1 evidence gaps (`T5-11`, `T5-36`, `T5-59`, `T5-41`) not dispositioned | process | §L | HIGH |
| **X-3** | H-1/H-2 dual truth with no reconciler (I-2 absent) | technical-design | §H, §I | HIGH |
| **X-4** | H-3 `rate_restrictions` epistemic conflict unresolved (E-1 unknown writer) | technical + business (O-2) | §H, G-8 | HIGH |
| **X-5** | Carry-overs BLK-1/BLK-2/Deviation C/D open | business (O-8) | `04:1410-1425` | HIGH |
| **X-6** | Flag sequence J-4…J-7 unscheduled; J-2 deploy invariant unsatisfied | operational (O-4, O-5) | §J | HIGH |
| **X-7** | No Phase 6 document set, stage model, or task ID space exists | process | §C | HIGH |

### P.3 Conditions observed as favourable

Verified by direct read: authority bound and live (`D-1`); single write port (`D-4`, two routes); read-only reconciliation pinned by test (`I-1`); legacy writer set from booking/release paths is empty in-repo (`M-8`); schema frozen; WS-N scan gates active on both the API and web sides (families T5-77…T5-86); rollback = one flag + one binding line; no admin/mobile surface to migrate.

---

## Conclusions

### Conclusion 1 — What is known

The availability authority is built, bound and in production use: `RESERVATION_AVAILABILITY_PORT → AvailabilityAssertionService` (`availability.module.ts:45`), a two-route read surface (`availability.controller.ts:13,21`), the snapshot/calculator, restriction evaluator and restriction write service, all consumed by Reservations, Front Office and holds (`D-9`). `gba.a3.authoritative` is ON (`.env:68`) and the A3 matrix therefore serves authority values. Legacy counter **writes from booking and release paths are empty in-repo** (`M-8`); the sole remaining legacy `availability` writer is `POST /rates/engine/modify → crs.modifyReservation → inventoryDomain.reserve/release` (`F-5/F-6`), deliberately retained and E-5-gated. Retired engine routes are 404 by construction and pinned by test. Phase 5 is closed with 12/12 exit gates PASS. Phase 6's written scope is known verbatim from `availability-phase3-implementation-plan.md:427,442,644-646`.

### Conclusion 2 — What is unknown

(E-5) which external systems call `/rates/engine/*` — no gateway, nginx or ops logs exist in the repository. (E-1) who writes `rate_restrictions` — no in-repo writer exists, so the authority cannot prove the store. Two Phase 5 tasks have **no execution record anywhere** (`T5-11`, `T5-36`), and `T5-41`/`T5-59` have no dedicated record — `T5-59`'s typed delete-reject is defined but never thrown (`L-7`). The runtime behaviour of the matrix room-type filter (N-1) was not executed in this audit, only read. Phase 6's own document set, stage structure and task-ID space do not exist (`C.4 D-4/D-5`).

### Conclusion 3 — The real Phase 6 problems

1. **The external modify path is an unaudited cross-domain write** (`H-1/N-2`): it mutates `reservations` and legacy counters in one transaction and never touches the assertion port, so every use desynchronises the two truths. This is precisely B-1's "removal of `crs-engine` inventory calls".
2. **There is no counter-vs-balance reconciler** (`I-2`) — the only availability reconciler compares a projection to a snapshot, is on-demand, repairs nothing, and persists nothing (`I-1/N-4`).
3. **There is no population-cutover tooling** (`I-3`) for `LEGACY → ASSERTION_MANAGED`, and bulk backfill is forbidden by D-21 (`04:438`).
4. **`rate_restrictions` is simultaneously authoritative and untrusted** (`H-3`), producing quote responses whose `blocked` and `available` fields come from different epistemic systems (`H-4`).
5. **A live defect on the authority surface** (`N-1`): supplying `roomType` to `GET /availability/matrix` interpolates a non-existent `rt` alias into an unconditional query, failing in both flag states; recorded by Phase 5 as not-fixed and never fixed.
6. **The legacy-pickup and GBA sequences are entirely dormant** — cascade, reconciliation, wash, two-layer consult, canonical read/write are all OFF, including a documented deploy invariant (`J-2/N-3`).

### Conclusion 4 — Work that is NOT required

No schema or migration authoring; no rebuild of the assertion/snapshot/evaluator/restriction-write/reconciliation modules; no re-pointing of Reservations, Front Office, holds or A3 restriction writes; no re-retirement of `engine/availability`, `engine/restrictions`, `engine/book`, `engine/release`, `rates/availability`; no re-do of T5-50 (repository legacy leg) or T5-43 (duplicate route pages); no admin or mobile availability surface (P-19/M-7); no deletion of the legacy `availability` table (P-16 → Phase 11); no Phase 5 re-audit; no GitHub operations. None of these should appear as Phase 6 tasks.

### Conclusion 5 — Items that are business decisions vs technical decisions

**Business / domain (10):** `room_inventory` disposition (O-1); `rate_restrictions` epistemic status (O-2); external `/rates/engine/modify` end-of-life (O-3); cascade deploy-invariant timing (O-4); canonical pickup soak definition and ordering (O-5); typed delete-reject vs raw FK (O-6); GBA client-math allowlist permanence (O-7); BLK-1/BLK-2 wash rules (O-8); occupancy-analytics contract (O-9); outbound channel publication (O-10).

**Technical (8):** matrix alias/parameterisation fix (O-11); `modifyReservation` port-call vs retirement (O-12); reconciliation persistence and cadence (O-13); population-cutover mechanism (O-14); `evaluateRestrictions` consult strategy (O-15); `/tax-rates` merge (O-16); Phase 3 register re-baselining (O-17); disposition of `T5-11`/`T5-36`/`T5-41`/`T5-59` (O-18).

**Unknown (1):** E-5 external consumer inventory — blocks O-3 and the timing of F-5/G-3 retirement.

### Conclusion 6 — Readiness to proceed

**Phase 6 is ready to be *documented and decided*, and not ready to be *executed*.** The scope (B-1…B-10), the boundary fence (§B), the as-built topology (§D), the legacy inventory with REMOVE/KEEP/DEFER classifications (§G), the divergence register (§H), the flag census (§J), the consumer inventory (§K), the contradiction register (§M) and the decision split (§O) are all established from evidence. Entry blockers X-1…X-7 remain: E-5 must be answered; the four unevidenced Phase 5 tasks must be dispositioned; the dual-truth and `rate_restrictions` conflicts must be decided; the Phase 4 carry-overs must be closed; and the flag sequence must be scheduled. Nothing in this audit requires a code change to begin that work.

### Conclusion 7 — Directive to stop

This audit is complete and self-contained at `docs/availability/phase-6/01_FORENSIC_AUDIT.md`. No code was modified, no file deleted, no schema or migration touched, no test executed, no GitHub operation performed, no decision recorded as made, no Phase 6 stage, task register, or document list invented beyond this single inherited-position artifact. **Stop here.** The next action belongs to a human: disposition §O and §P.2, then author the Phase 6 plan.

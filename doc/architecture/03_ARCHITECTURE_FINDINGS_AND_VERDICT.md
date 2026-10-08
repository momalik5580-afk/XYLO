# Document 03 — Architecture Findings and Verdict

**Step 03 of the XYLO Project-Wide Architecture Investigation**

Status: COMPLETE
Follows: `docs/architecture/01_CURRENT_ARCHITECTURE_AUDIT.md` (Document 01, Step 01) and `docs/architecture/02_OPEN_EVIDENCE_RESOLUTION.md` (Document 02, Step 02)
Date: 2026-10-08

---

## 1. Document Scope & Constraints

This document is the **judgement step** of the investigation. It reconciles Document 01 (the finding register) with Document 02 (the resolved evidence) into one final, classified register, and then issues three verdicts: backend repairability, major-domain readiness, and overall architecture.

In scope:

- Reconciliation of both inputs into a single final position for each of the 53 findings.
- Reclassification of **every** row — including the three rows that entered this step as `UNCERTAIN / EVIDENCE REQUIRED`.
- Separation of true architectural problems from technical debt, legacy residue, implementation bugs, and sound design.
- An explicit backend repairability verdict.
- An assessment of where architectural risk concentrates.
- An explicit major-domain readiness verdict.
- Architectural preconditions, stated as **conditions that must hold**, not as work items.
- An explicit overall architecture verdict.
- Residual uncertainties and explicit non-decisions.

Out of scope (explicitly not done in this step):

- **No source code, database schema, migration, API, CI configuration, deployment manifest, runtime configuration, or frontend behaviour was created or modified.** No new evidence was gathered in this step; Documents 01 and 02 are the sole evidentiary basis, and every fact cited here is attributable to one of them.
- No target architecture, no future-state design, no gap analysis.
- No correction plan, remediation sequence, workstream, task breakdown, owner, or effort estimate.
- Documents 01 and 02 are **not** modified, renumbered, or reinterpreted.
- **No new finding IDs are created.** Document 02 §16 observations are used only as evidence supporting existing rows or readiness judgements; they are never given an `ARCH-` identifier here.
- The **Reservations** rebuild is out of scope. The Reservations domain is referenced only where it is load-bearing for another major domain's readiness (principally front-office), and never as a subject of assessment.

---

## 2. Authoritative Inputs

| Input | Size | Role in this document |
|---|---|---|
| `docs/architecture/01_CURRENT_ARCHITECTURE_AUDIT.md` | 90,126 bytes, 841 lines | Source of the 53-finding register (§13), the classification vocabulary (§1.3), the repairability buckets (§14), the backend impact matrix (§15), and the original verdict (§16) |
| `docs/architecture/02_OPEN_EVIDENCE_RESOLUTION.md` | 83,840 bytes, 1,150 lines | Source of the resolved evidence for all ten open items (E-01…E-10), the per-finding impact statements (§15), and the residual/new evidence list (§16) |

Nothing else is authoritative. Repository files, the live database, framework source, CI configuration, and Kubernetes manifests were read **during Steps 01 and 02**; they are quoted here only as Document 01 or Document 02 recorded them. Where this document states a number, that number carries the document reference it came from.

Both inputs were read in full before this document was drafted. Neither input was edited; Document 01 remains at its Step 01 timestamp and Document 02 at its Step 02 timestamp.

---

## 3. Method

### 3.1 Operations performed

1. **Absorb** every Document 02 §15 impact statement into the affected register row.
2. **Assign** a final classification, a severity, an evidence status, a backend impact, and a blocker status to all 53 rows — none omitted, none inferred from silence.
3. **Group** the final classifications into themes to identify which findings are structural defects and which merely resemble them.
4. **Judge** repairability, risk concentration, per-domain readiness, preconditions, and the overall verdict.

### 3.2 Vocabulary (used verbatim)

**Final classification** (Document 01 §1.3, unchanged):

| Value | Meaning |
|---|---|
| `SOUND` | Working as intended; evidence supports keeping the design |
| `ARCHITECTURAL PROBLEM` | Structural defect in boundaries, ownership, layering, or system shape |
| `TECHNICAL DEBT` | Recognised shortcut/duplication that does not by itself break the structure |
| `LEGACY RESIDUE` | Superseded code/data that still exists and still affects behaviour |
| `IMPLEMENTATION BUG` | Incorrect implementation of an otherwise correct design decision |
| `UNCERTAIN / EVIDENCE REQUIRED` | Cannot be settled from repository evidence alone |

**Severity** — assigned to every row in this document (Document 01 assigned it only to `ARCHITECTURAL PROBLEM` rows, with two noted exceptions): `CRITICAL` | `HIGH` | `MEDIUM` | `LOW` | `NONE`. `NONE` is used for every `SOUND` row.

**Evidence status** (defined here, because neither input defined a status vocabulary):

| Value | Meaning |
|---|---|
| `RESOLVED (E-xx)` | The row entered this step as `UNCERTAIN`; Document 02 supplied the evidence that closes it |
| `CONFIRMED (E-xx)` | Document 02 evidence directly substantiates the row as written |
| `REINFORCED (E-xx)` | Document 02 evidence strengthens the row without restating it |
| `REFINED (E-xx)` | Document 02 evidence preserves the row's class but supersedes part of its wording |
| `RE-CHARACTERISED (E-xx)` | Document 02 evidence preserves the row's class but changes what the row is understood to mean |
| `UNAFFECTED` | Document 02 did not address the row |

**Architectural?** — `Yes` if and only if the final classification is `ARCHITECTURAL PROBLEM`; `No` otherwise. The column is deliberately mechanical: whether a `LEGACY RESIDUE` or `TECHNICAL DEBT` row still carries structural *consequence* is expressed in its rationale and in §12, not in this flag.

**Backend impact** (Document 01 §15 categories): `None` | `Internal` | `API` | `Database` | `Database + API` | `Deployment/config` | `Unknown`.

**Major-domain blocker?**

| Value | Meaning |
|---|---|
| `YES` | A global precondition — the finding must be satisfied before *any* major domain extends (§14, P1–P4) |
| `CONDITIONAL` | A precondition only for the scope named in the rationale (§14, P5–P11; §13 basis column) |
| `NO` | Not a precondition for major-domain work, whatever else it requires |

**Repairability** (Document 03 Step 03 vocabulary): `HIGH — repairable in place` | `MODERATE — repairable with controlled migrations` | `LOW — major restructuring required` | `NOT DETERMINABLE`.

**Readiness**: `READY` | `READY WITH ARCHITECTURAL CONSTRAINTS` | `NOT READY — ARCHITECTURAL CORRECTION REQUIRED FIRST` | `NOT DETERMINABLE`.

### 3.3 Rules applied

- **Reclassification rule.** A final classification is changed only where the resolved evidence or a clearer application of the Document 01 §1.3 definitions warrants it. Confirmation alone does not move a row.
- **Double-counting rule.** Where a resolved row proves to be co-extensive with an existing row, it is classified in that row's class, marked *absorbed* in its rationale, and **not** counted as an additional blocker or as an additional distinct structural defect. Two rows are affected by this rule: ARCH-051 (absorbed into ARCH-008) and ARCH-052 (absorbed into ARCH-041).
- **No-new-findings rule.** Document 02 §16.2 records fourteen newly observed facts. None receives an `ARCH-` identifier. Where one of them bears on a readiness verdict, the readiness column cites it explicitly as Document 02 evidence and says so.
- **Language constraint.** This document states conditions and current-state judgements. It does not prescribe relocation, substitution, combination, removal, or re-implementation of any component, and it does not prescribe the order in which anything would occur.

---

## 4. Evidence Reconciliation

### 4.1 The sixteen findings the evidence bears on

| ID | Document 01 position | Document 02 evidence | Reconciled position | Effect on classification / severity / blocker |
|---|---|---|---|---|
| **ARCH-007** | `ARCHITECTURAL PROBLEM`, Critical — tenant scope taken from a client-controllable `x-property-id` header; property authorization explicitly disabled; no equivalence check against `x-tenant-id` | E-01: database-level isolation is inert, so header-based scoping is effectively the sole tenant boundary in the request path. Document 02 §16.5: when `request.user` is absent, both guards fall back to client-supplied `x-property-id` / `x-tenant-id` | Retained, Critical, and **reinforced**: the header is not one of several boundary mechanisms, it is the only one that operates | Unchanged / Critical / **YES** |
| **ARCH-008** | `ARCHITECTURAL PROBLEM`, Critical — isolation not reproducible from source (0 enabling statements, 1 policy, missing migration, dead extension, unregistered middleware) | E-01: 386 RLS-enabled tables and 392 policies live; the repository holds 1 policy file and 0 enabling statements, and that one policy is **absent live**; the connecting role is table owner with `SUPERUSER` + `BYPASSRLS`, 0 `FORCE`; the only setters write `app.current_tenant` while all 375 policies read `app.hotel_id` | Retained, Critical, **sharpened in both directions**: the layer is 392 policies deep and unreproducible; and it is simultaneously inert | Unchanged / Critical / **YES** |
| **ARCH-009** | `ARCHITECTURAL PROBLEM`, High — three Prisma schemas, 1,018 models, one physical database, one migration owner | E-05: 1,018 models map to 1,018 unique tables; live has 1,033 tables; **0 modelled tables missing**; 14 live tables unmodelled | Unchanged; E-05 adds that the three-way split is intact and that the drift runs one way only | Unchanged / High / NO |
| **ARCH-010** | `ARCHITECTURAL PROBLEM`, High — migration state unrecoverable from git | E-05: 51 directories on disk, **23 tracked**, 28 untracked, no `.gitignore` rule; 51/51 ledger match but 2 stale `finished_at IS NULL` rows; `email_outbox` applied then dropped outside history; `person_discrepancies` referenced 15× and never created; `companion.sql` outside the runner | Confirmed and quantified. The nature is refined: the defect is not a dirty schema file but that **version control does not contain the schema history the live database has applied, and the ledger has diverged from the live database** | Unchanged / High / **NO → YES** (elevated to a global precondition on this evidence) |
| **ARCH-011** | `ARCHITECTURAL PROBLEM`, High — two complete inventory pipelines, no bridge | E-06: the second pipeline's client is imported at 20+ production sites; it is wired, not vestigial | Retained and reinforced | Unchanged / High / **CONDITIONAL** (inventory, purchasing) |
| **ARCH-016** | `ARCHITECTURAL PROBLEM`, High — nine unsequenced global guards, two permission vocabularies | E-03: order derived from NestJS 10.4.22 source — `JwtAuthGuard` first, then property/tenant, then five authorization guards, then the platform permission guard, then scoped `@UseGuards`; downstream guards fail **closed**; the two permission guards execute on every request over disjoint keys (47 vs 173 sites), both returning `true` when their decorator is absent | Retained and precisely specified. *"Unsequenced"* is superseded by *"sequenced, and the sequence is safe"*; *"two vocabularies"* is confirmed and quantified | Unchanged / High / **CONDITIONAL** (new API surface) |
| **ARCH-024** | `ARCHITECTURAL PROBLEM`, High — test architecture blind spots | E-10: `AVAILABILITY_TEST_DATABASE_URL` appears in zero workflows; `ci-test.yml` has no `services:` and no `env:`; a spec asserts the *documentation string* rather than the setting | Reinforced with exact mechanism: the gap is structural, not an oversight in one workflow | Unchanged / High / **CONDITIONAL** (behavioural change in the 26 untested modules) |
| **ARCH-028** | `ARCHITECTURAL PROBLEM`, Medium — platform surface registered and guarded, zero consumers | E-07: zero first-party references across web/admin/mobile/packages; no rewrite routes there; and per E-04 no reachable path exists without hard-coding the doubled form | Confirmed; the surface is simultaneously unreachable at its advertised path and unused | Unchanged / Medium / NO |
| **ARCH-036** | `TECHNICAL DEBT`, Medium — admin and mobile appear to be prototypes | E-08: admin has an image, a manifest, and CI coverage but exactly **1** API call site; mobile has **6** call sites and no Dockerfile, no image matrix entry, no manifest, no workflow | Confirmed and split into two sub-facts with opposite profiles (deployed-but-disconnected vs connected-but-undeployed) | Unchanged / Medium / NO |
| **ARCH-041** | `LEGACY RESIDUE`, High — legacy `availability` counter with no application writer | E-02: two database-resident writers exist and are absent from every repository file; both sit on tables the application has all-but abandoned (`allotment_pickup` = 0 rows, `allotment` = 1); the active pipeline writes other tables; no scheduled writer exists (`pg_cron` absent) | Re-characterised: **dormant in traffic, live in DDL**. The dual-writer risk is not currently exercised, and the coupling is not severed | Unchanged / High / NO |
| **ARCH-046** | `IMPLEMENTATION BUG`, Critical — stale `file:`-linked inventory client | E-06: live `property_id` is `uuid` on all 17 columns; the source schema agrees (`@db.Uuid`); the **installed** client disagrees (`VarChar(20)`) and is 4 models behind; 20+ import sites; runtime-error sub-question `UNABLE TO VERIFY` | Confirmed and re-anchored: source schema and database agree; the generated client is the divergent artifact | Unchanged / Critical / **CONDITIONAL** (inventory) |
| **ARCH-048** | `IMPLEMENTATION BUG`, Medium — doubled `api/v1` prefix on 21 controllers | E-04: the installed framework concatenates unconditionally (`route-path-factory.js`), with no exclusion and no versioning; 0 references to either path form outside Document 01 | Confirmed at framework-behaviour level; upgraded from "static analysis says" to "framework source says" | Unchanged / Medium / NO |
| **ARCH-050** | `IMPLEMENTATION BUG`, Medium — worker polls a table removed by migration `20260706000000` | E-05: the table is absent live, no migration in any of the 51 directories drops it, the drop came from a hand-run script, and `EMAIL_OUTBOX_DISABLED` is absent from the Kubernetes manifest — so the worker is enabled on any deployment built from `infra/k8s` | Confirmed with one fact corrected: the operative gap is the deployment manifest, not only `.env.example` | Unchanged / Medium / NO |
| **ARCH-051** | `UNCERTAIN / EVIDENCE REQUIRED` | E-01: `RESOLVED` — RLS is enabled on 386 tables with 392 policies and the repository reproduces essentially none of it | Uncertainty closed. The resolved fact is the same structural defect already carried by ARCH-008 | **Changed → `ARCHITECTURAL PROBLEM` / MEDIUM / absorbed into ARCH-008 / NO** |
| **ARCH-052** | `UNCERTAIN / EVIDENCE REQUIRED` | E-02: `RESOLVED` — the mutators are `trg_update_allotment_pickup` → `update_allotment_pickup()` and `trg_block_availability` → `block_availability_on_allotment()`, plus a timestamp trigger; all absent from the repository | Uncertainty closed. The resolved fact is superseded machinery that still exists and could still fire, on tables the application has abandoned — the definition of `LEGACY RESIDUE`, and the same subject as ARCH-041 | **Changed → `LEGACY RESIDUE` / MEDIUM / absorbed into ARCH-041 / NO** |
| **ARCH-053** | `UNCERTAIN / EVIDENCE REQUIRED` | E-03: `RESOLVED` — order derived from framework source; `JwtAuthGuard` runs first; downstream guards are fail-closed; no ordering-driven under-guarding is possible | Uncertainty closed **in the safe direction**: the specific risk Document 01 named did not materialise. Double evaluation and opt-in semantics remain, and are already carried by ARCH-016 | **Changed → `SOUND` / NONE / residual carried by ARCH-016 / NO** |

### 4.2 What the three resolutions mean

- **ARCH-051 confirms rather than mitigates.** Document 01 asked whether the live database's isolation layer exists. It does — at a scale (386 tables, 392 policies) larger than the 375 introspected models suggested — and it cannot be rebuilt, reviewed, or migrated from this repository. The row therefore joins ARCH-008 rather than relieving it. Its severity is set to `MEDIUM` as an absorbed row so that the Critical mass is counted once.
- **ARCH-052 splits the difference Document 01 posed.** The choice was "dormant residue **or** a live database-level writer". The evidence selects neither pole: the writers are live objects and their source tables are not written by the application. The tie-break is table activity, and by that test the row is residue, not an active architectural defect. Its severity is `MEDIUM` as an absorbed row.
- **ARCH-053 is the one uncertainty that resolved favourably.** Authentication precedes every authorisation enhancer and the authorisation guards fail closed, so the ordering risk is disproven. What survives — two permission guards evaluating on every request, both inert when their decorator is absent — is Document 01's ARCH-016, unchanged. Nothing is lost by closing the row as `SOUND`.

### 4.3 What the evidence did not change

- **36 of 53 rows are `UNAFFECTED`** by Document 02. Confirmation is not reclassification.
- **No row was downgraded out of `ARCHITECTURAL PROBLEM`, and no row was upgraded into it**, other than ARCH-051 via the double-counting rule. Document 01's classification of the 26 original structural defects stands on the evidence.
- **ARCH-009 and ARCH-032 were expressly untouched** by the ten questions and remain as recorded.
- **One framing was corrected in Document 01's favour and one against it.** The table-level schema baseline is *cleaner* than "8,793 uncommitted lines" implied — every modelled table exists live. The version-control position is *worse* than recorded — 28 of 51 migrations are untracked and the ledger has diverged from the live database in two known places.
- **One prior impression was softened:** E-09 shows the `events` queue's producer→consumer leg is live and has completed jobs. ARCH-018's specific claims (no in-process handler registrations, a second outbox processor with no callers, an unregistered analytics consumer) are unaffected; the fragmentation is real, but not every path is dead.

---

## 5. Finding Classification Summary

### 5.1 Class totals

| Classification | Document 01 | Document 03 | Change |
|---|---:|---:|---|
| `SOUND` | 6 | **7** | +1 (ARCH-053) |
| `ARCHITECTURAL PROBLEM` | 26 | **27** | +1 (ARCH-051, absorbed into ARCH-008) |
| `TECHNICAL DEBT` | 8 | **8** | — |
| `LEGACY RESIDUE` | 5 | **6** | +1 (ARCH-052, absorbed into ARCH-041) |
| `IMPLEMENTATION BUG` | 5 | **5** | — |
| `UNCERTAIN / EVIDENCE REQUIRED` | 3 | **0** | −3 (all resolved) |
| **Total** | **53** | **53** | none omitted, none added |

The register contains **26 distinct structural defects plus 1 absorbed duplicate**. The count of distinct structural defects is unchanged from Document 01.

### 5.2 Severity distribution (all 53 rows)

| Severity | Count | Composition |
|---|---:|---|
| `CRITICAL` | 4 | ARCH-007, ARCH-008 (structural) · ARCH-046, ARCH-047 (implementation) |
| `HIGH` | 17 | 16 structural (ARCH-009…ARCH-024) + ARCH-041 (legacy residue) |
| `MEDIUM` | 24 | 9 structural (ARCH-025…ARCH-032, ARCH-051) · 8 debt · 4 legacy · 3 bugs |
| `LOW` | 1 | ARCH-045 |
| `NONE` | 7 | ARCH-001…ARCH-006, ARCH-053 |

The Critical ratio is unchanged: **2 of 4 Critical findings are structural (tenancy), 2 are implementation defects that are in-place correctable.**

### 5.3 Evidence status distribution

| Evidence status | Count |
|---|---:|
| `UNAFFECTED` | 36 |
| `CONFIRMED` | 7 |
| `REINFORCED` | 4 |
| `REFINED` | 2 |
| `RE-CHARACTERISED` | 1 |
| `RESOLVED` | 3 |

### 5.4 Consolidated register

Severity is assigned to every row in this document; Document 01 assigned it only to `ARCHITECTURAL PROBLEM` rows (with ARCH-041 and ARCH-046/047 as recorded deviations, whose values are retained here).

| ID | Original Classification | Final Classification | Severity | Evidence Status | Architectural? | Backend Impact | Major-Domain Blocker? | Rationale |
|---|---|---|---|---|---|---|---|---|
| ARCH-001 | SOUND | SOUND | NONE | UNAFFECTED | No | None | NO | Coherent workspace and single composition root; `platform`/`common`/`core` have zero edges into feature modules. |
| ARCH-002 | SOUND | SOUND | NONE | UNAFFECTED | No | None | NO | Rebuilt domains implement genuine DDD — the strongest asset and the in-repo reference pattern. |
| ARCH-003 | SOUND | SOUND | NONE | UNAFFECTED | No | None | NO | Custom CQRS bus with a five-pipe pipeline and duplicate-registration guard; 188 handlers, correct command boundary. |
| ARCH-004 | SOUND | SOUND | NONE | UNAFFECTED | No | None | NO | Transactional assertion engine plus a disposable-schema Postgres harness with real migration replay. |
| ARCH-005 | SOUND | SOUND | NONE | UNAFFECTED | No | None | NO | OTel, pino, request logging, and exception filtering wired once, globally. |
| ARCH-006 | SOUND | SOUND | NONE | UNAFFECTED | No | None | NO | OTA ingress live with HMAC verification, dedupe, and creation through the reservations facade. |
| ARCH-007 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | CRITICAL | REINFORCED (E-01) | Yes | Internal | **YES** | Tenant scope comes from a client-controllable header with property authorization disabled; with database isolation inert this is the only boundary that operates. |
| ARCH-008 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | CRITICAL | CONFIRMED (E-01) | Yes | Database | **YES** | 386 live RLS tables and 392 policies unreproducible from source; role is owner + `SUPERUSER` + `BYPASSRLS`, 0 `FORCE`, GUC never set and misspelled by its setters — no effective database isolation in any environment this repository can produce. |
| ARCH-009 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | Database | NO | 1,018 models over one physical database across three schemas with a single migration owner; cross-schema keys remain untyped strings. |
| ARCH-010 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | CONFIRMED (E-05) | Yes | Database | **YES** | 28 of 51 migrations untracked, 2 unfinished ledger rows, one table dropped outside history, one referenced 15× and never created — no reproducible schema baseline. |
| ARCH-011 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | REINFORCED (E-06) | Yes | Database + API | CONDITIONAL (inventory, purchasing) | Two unlinked inventory worlds; the second client is imported at 20+ production sites, so the pipeline is wired rather than vestigial. |
| ARCH-012 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | Database | CONDITIONAL (guest-identity work) | Dual reservation and triple guest sources of truth with a documented dual-write; any guest-identity work must reconcile two models. |
| ARCH-013 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | Internal | **YES** | 78 cross-module internal-import edges; front-office reaches seven internal layers of reservations across 35 edges — module boundaries are nominal. |
| ARCH-014 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | Internal | NO | The documented shared repository layer has 2 references; 233 module files reach `PrismaService`, 144 use raw SQL, 6 controllers inject it directly. |
| ARCH-015 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | Internal | NO | Two CQRS stacks expose identically named tokens; 31 of 37 modules use neither; a wrong import compiles and routes silently. |
| ARCH-016 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | REFINED (E-03) | Yes | API | CONDITIONAL (new API surface) | Sequence is safe (JWT first, fail-closed), but two permission vocabularies (47 vs 173 sites) execute on every request, are disjoint, and are both inert when their decorator is absent. |
| ARCH-017 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | Internal + API | NO | Two to five parallel implementations per cross-cutting concern — 5 Prisma services, 2 audit systems, 2 policy engines, 3 idempotency paths, 2 `AppException` classes in one directory. |
| ARCH-018 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | REFINED (E-09) | Yes | Internal | NO | In-process dispatcher has zero handler registrations and always no-ops; a second outbox processor has no callers; the analytics consumer is unregistered. E-09 shows only the BullMQ `events` leg carries traffic. |
| ARCH-019 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | Internal | CONDITIONAL (money/inventory writes) | Check-in and check-out post money-affecting SQL outside the command transaction; four services perform multi-step writes with no transaction at all. |
| ARCH-020 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | Database | NO | Two services execute `CREATE TABLE`/`CREATE INDEX` during application boot; schema changes occur outside migration ownership. |
| ARCH-021 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | None | NO | Four HTTP clients, three `ApiError` classes, five query-key registries with incompatible shapes for the same root key — no single data-access contract. |
| ARCH-022 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | None | NO | Three competing organizational schemes, a legacy⇄feature import cycle, and 19 `app/` files reaching into feature internals. |
| ARCH-023 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | UNAFFECTED | Yes | None | NO | 13 of 20 global stores fetch server state; the same eight endpoints are cached twice with no shared invalidation; three independent `Reservation` types. |
| ARCH-024 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | HIGH | REINFORCED (E-10) | Yes | None | CONDITIONAL (behavioural change in 26 untested modules) | DB-gated suites skip on every PR (no service, no env); 26 of 37 modules untested; no e2e suite; no type gate over 222 api tests. |
| ARCH-025 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | MEDIUM | UNAFFECTED | Yes | Internal | CONDITIONAL (folio/billing work) | Check-out writes folio data directly and imports billing while billing itself is a three-file module that cannot enforce its own invariants. |
| ARCH-026 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | MEDIUM | UNAFFECTED | Yes | API | NO | Two owners for property and two for company, on different Prisma clients with duplicate DTOs — two writes to one concept land in two tables. |
| ARCH-027 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | MEDIUM | UNAFFECTED | Yes | None | NO | 19 of 37 modules are stub or thin, 37 of 55 inventory sub-directories are empty, 2 modules are unregistered — module count overstates capability. |
| ARCH-028 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | MEDIUM | CONFIRMED (E-07) | Yes | API | NO | A surface carrying 173 authorization annotations, with zero first-party consumers and no reachable path (routes doubled per E-04). |
| ARCH-029 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | MEDIUM | UNAFFECTED | Yes | Internal | NO | One real `forwardRef` cycle (Availability⇄RatesInventory), one latent cycle masked by an unregistered module, one unbalanced forward reference. |
| ARCH-030 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | MEDIUM | UNAFFECTED | Yes | Internal | NO | A reverse edge from `infrastructure` into `modules`; a global `SharedModule` registers feature modules behind a stale "no cycle" comment. |
| ARCH-031 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | MEDIUM | UNAFFECTED | Yes | None | NO | No boundary rules anywhere; lint covers only `app components lib`, leaving 612 of 871 web files unlinted. |
| ARCH-032 | ARCHITECTURAL PROBLEM | ARCHITECTURAL PROBLEM | MEDIUM | UNAFFECTED | Yes | None | NO | Three overlapping Temporal layers; exactly one workflow, never registered; the worker loads an empty workflow barrel; queues nothing listens on. |
| ARCH-033 | TECHNICAL DEBT | TECHNICAL DEBT | MEDIUM | UNAFFECTED | No | API | NO | 30 of 87 controllers accept `@Body() body: any`; two parallel reservation DTO sets are both consumed. |
| ARCH-034 | TECHNICAL DEBT | TECHNICAL DEBT | MEDIUM | UNAFFECTED | No | None | NO | 46 components exceed 500 lines with business rules inside them; the shared pricing module has no consumers. |
| ARCH-035 | TECHNICAL DEBT | TECHNICAL DEBT | MEDIUM | UNAFFECTED | No | None | NO | Nine hardcoded `localhost` fallbacks, three `readCookie` copies, duplicated auth logic across web and admin. |
| ARCH-036 | TECHNICAL DEBT | TECHNICAL DEBT | MEDIUM | CONFIRMED (E-08) | No | Unknown | NO | Admin is deployed with exactly 1 API call site; mobile has 6 call sites and no image, manifest, or workflow — two prototypes with opposite profiles. |
| ARCH-037 | TECHNICAL DEBT | TECHNICAL DEBT | MEDIUM | UNAFFECTED | No | None | NO | Two `ConfigModule` roots, an empty `config/` directory, an unimported `@xylo/config`, two feature-flag systems, direct `process.env`. |
| ARCH-038 | TECHNICAL DEBT | TECHNICAL DEBT | MEDIUM | UNAFFECTED | No | Internal | NO | Payment gateway stub hardwired, push SDKs absent, channel metrics fabricated — operators cannot separate live from simulated behaviour. |
| ARCH-039 | TECHNICAL DEBT | TECHNICAL DEBT | MEDIUM | UNAFFECTED | No | None | NO | Documentation describes a Temporal check-in pipeline and a repository convention the code does not have; api lint is `echo 'ok'`. |
| ARCH-040 | TECHNICAL DEBT | TECHNICAL DEBT | MEDIUM | UNAFFECTED | No | None | NO | api tests are excluded from typecheck and transpiled with diagnostics off; one test already imports a path that does not exist. |
| ARCH-041 | LEGACY RESIDUE | LEGACY RESIDUE | HIGH | RE-CHARACTERISED (E-02) | No | Database | NO | Two database-resident writers exist, are absent from every repository file, and sit on tables the application has abandoned — dormant in traffic, live in DDL. |
| ARCH-042 | LEGACY RESIDUE | LEGACY RESIDUE | MEDIUM | UNAFFECTED | No | Internal | NO | 82 files have zero non-test importers, including all three Reservations Phase-6 port adapters and seven handlers that would fail if routed. |
| ARCH-043 | LEGACY RESIDUE | LEGACY RESIDUE | MEDIUM | UNAFFECTED | No | Database + API | CONDITIONAL (inventory, purchasing) | The legacy `inventory_*` pipeline is still written by purchasing, procurement, and housekeeping alongside `xylo_inventory`, with no bridge and no reconciliation. |
| ARCH-044 | LEGACY RESIDUE | LEGACY RESIDUE | MEDIUM | REINFORCED (E-05) | No | Database | NO | 14 live tables have no Prisma model — including `folios`, `folio_postings`, `currencies`, `fx_rates`, `routing_instructions` — so critical money tables sit outside the ORM. |
| ARCH-045 | LEGACY RESIDUE | LEGACY RESIDUE | LOW | UNAFFECTED | No | None | NO | 28 tracked-but-deleted files, orphan SQL migration files, tracked build artifacts, 40 stale compiled specs in `dist`. |
| ARCH-046 | IMPLEMENTATION BUG | IMPLEMENTATION BUG | CRITICAL | CONFIRMED (E-06) | No | Database | CONDITIONAL (inventory) | Live database and source schema both say `uuid`; the installed `file:`-linked client says `VarChar(20)` and is 4 models behind, at 20+ import sites. |
| ARCH-047 | IMPLEMENTATION BUG | IMPLEMENTATION BUG | CRITICAL | UNAFFECTED | No | Deployment/config | NO | The JWT signing secret is inlined into the client bundle with hardcoded fallbacks — token integrity for the web app is void. |
| ARCH-048 | IMPLEMENTATION BUG | IMPLEMENTATION BUG | MEDIUM | CONFIRMED (E-04) | No | API | NO | The framework concatenates unconditionally, so 21 platform controllers resolve at `/api/v1/api/v1/...`; no consumer uses either path form. |
| ARCH-049 | IMPLEMENTATION BUG | IMPLEMENTATION BUG | MEDIUM | UNAFFECTED | No | None | NO | The documented e2e command points at a config file and a directory that do not exist. |
| ARCH-050 | IMPLEMENTATION BUG | IMPLEMENTATION BUG | MEDIUM | CONFIRMED (E-05) | No | Internal | NO | The target table is absent live, no migration drops it, and the disable flag is absent from the Kubernetes manifest — so the worker is enabled on deployment. |
| ARCH-051 | UNCERTAIN / EVIDENCE REQUIRED | ARCHITECTURAL PROBLEM | MEDIUM | RESOLVED (E-01) | Yes | Database | NO | Resolved: live RLS exists at scale and is unreproducible — co-extensive with ARCH-008. Absorbed; counted once, not a separate blocker. |
| ARCH-052 | UNCERTAIN / EVIDENCE REQUIRED | LEGACY RESIDUE | MEDIUM | RESOLVED (E-02) | No | Database | NO | Resolved: the mutators are two repository-absent triggers on abandoned tables — co-extensive with ARCH-041. Absorbed; counted once. |
| ARCH-053 | UNCERTAIN / EVIDENCE REQUIRED | SOUND | NONE | RESOLVED (E-03) | No | None | NO | Resolved safely: authentication precedes all authorisation and guards fail closed, so ordering cannot under-guard a route; residual double-evaluation is carried by ARCH-016. |

ARCH-021, ARCH-022, ARCH-023, and ARCH-031 are frontend-structure defects and therefore carry `Architectural? = Yes` under the mechanical rule of §3.2, while their backend impact remains `None` — frontend corrections carry no backend risk (Document 01 §14.2.3: 0 production imports of `apps/api` or `packages/db`).

**Register totals: 53 findings — 7 SOUND, 27 ARCHITECTURAL PROBLEM (26 distinct + 1 absorbed), 8 TECHNICAL DEBT, 6 LEGACY RESIDUE (5 + 1 absorbed), 5 IMPLEMENTATION BUG, 0 UNCERTAIN.**

---

## 6. True Architectural Problems

### 6.1 The test applied

A finding is a *true* architectural problem when correcting the local code would leave the structure wrong — that is, when the defect lies in a boundary, an ownership relation, a layer direction, or the shape of the system rather than in a particular implementation of a sound decision. Technical debt and implementation bugs can each be corrected without changing any structural relation. Legacy residue is superseded rather than misdesigned. Sound rows contain no defect at all.

By that test, **27 of 53 rows are `ARCHITECTURAL PROBLEM`: 26 distinct structural defects plus ARCH-051, which is absorbed into ARCH-008 and counted once.** They fall into ten themes.

### 6.2 The ten themes

| # | Theme | Rows | Distinct |
|---|---|---|---:|
| A | Tenant isolation has no authoritative source | ARCH-007, ARCH-008 (ARCH-051 absorbed) | 2 |
| B | Schema and migration ownership | ARCH-009, ARCH-010, ARCH-020 | 3 |
| C | Duplicated data ownership | ARCH-011, ARCH-012, ARCH-025, ARCH-026 | 4 |
| D | Module boundaries and layer direction | ARCH-013, ARCH-014, ARCH-029, ARCH-030 | 4 |
| E | Parallel cross-cutting subsystems | ARCH-015, ARCH-016, ARCH-017 | 3 |
| F | Communication paths that carry nothing | ARCH-018 | 1 |
| G | Transaction boundaries | ARCH-019 | 1 |
| H | Frontend structure | ARCH-021, ARCH-022, ARCH-023, ARCH-031 | 4 |
| I | Verification structure | ARCH-024 | 1 |
| J | Scaffolding and inert infrastructure | ARCH-027, ARCH-028, ARCH-032 | 3 |
| | **Total** | | **26** |

### 6.3 Theme A — Tenant isolation has no authoritative source (ARCH-007, ARCH-008)

This is the only theme containing both Critical structural findings, and the evidence makes it sharper rather than softer.

- **Who is authoritative for tenant scope?** A client-supplied header. `x-property-id` overrides the authenticated user's tenant, property authorization is explicitly disabled, and there is no equivalence check of the kind that exists for `x-tenant-id`. Document 02 added that when a request is unauthenticated, both guards fall back to client-supplied headers anyway.
- **What enforces scope below the application?** Nothing effective. Document 02 established that 386 live tables carry RLS with 392 policies, that the repository reproduces none of it (0 enabling statements, 1 policy file whose policy is absent live), that the only setters write a differently-named setting than the policies read, and that the connecting role owns every table with `SUPERUSER` + `BYPASSRLS` and no `FORCE`.

The conjunction is the structural fact: **in the live database the isolation layer exists but is inert, and in any environment this repository can produce it does not exist at all.** There is therefore no environment in which database-level tenant isolation currently holds, and the application-level boundary is a client-controllable header. Neither half alone is a coding defect; together they describe a system whose tenant boundary is not located anywhere the system owns.

ARCH-051 is the evidence form of this same defect and is absorbed here.

### 6.4 Theme B — Schema and migration ownership (ARCH-009, ARCH-010, ARCH-020)

Three rows describe one condition: **the database is not wholly owned by the repository's declared schema process.**

- ARCH-009: 1,018 models across three schemas and three datasources over one physical database, with migration ownership held by only one schema and cross-schema references expressed as untyped strings.
- ARCH-010: as quantified by E-05, 28 of 51 migration directories are untracked; two ledger rows are unfinished duplicates; `email_outbox` was applied and then dropped outside migration history; `person_discrepancies` is referenced 15 times in production code and created nowhere; one migration's `companion.sql` lies outside the runner's execution rule.
- ARCH-020: two services execute DDL during application boot, which is the same condition expressed at runtime.

The drift direction matters and was refined by the evidence: **0 modelled tables are missing live**, so the schema files are not behind the database; the defect is that version control does not contain the schema history the database has applied, that 14 live tables are invisible to the schema, and that the ledger and the database have already diverged.

### 6.5 Theme C — Duplicated data ownership (ARCH-011, ARCH-012, ARCH-025, ARCH-026)

Four rows where one business concept has more than one authoritative store: two inventory worlds with no bridge (reinforced by E-06 — the second client is imported at more than 20 production sites); two reservation models and three guest models with a documented dual-write; folio data written from two modules while the owning module is a three-file shell; and two owners each for property and company on different Prisma clients.

This theme is architectural rather than legacy because both sides of each pair are **live and intended** — neither is superseded residue awaiting retirement. That distinction is what separates it from §8.

### 6.6 Theme D — Module boundaries and layer direction (ARCH-013, ARCH-014, ARCH-029, ARCH-030)

Boundaries exist as directory structure and are not enforced as relations:

- 78 production cross-module internal-import edges, of which front-office→reservations alone is 35 edges reaching seven internal layers (ARCH-013) — the single most compounding row in the register after tenancy.
- The documented shared repository layer has 2 references while 233 module files reach `PrismaService` and 144 use raw SQL (ARCH-014) — the stated layering is not the actual layering.
- One real module cycle, one latent cycle masked by an unregistered module, one unbalanced forward reference (ARCH-029).
- One true reverse edge from `infrastructure` into `modules`, and a global module that registers feature modules behind a stale comment (ARCH-030).

Document 01's observation stands: the direction of dependencies is mostly healthy; the defect is *depth*, not *direction*.

### 6.7 Theme E — Parallel cross-cutting subsystems (ARCH-015, ARCH-016, ARCH-017)

Every cross-cutting concern has two to five answers: two CQRS stacks with identically named tokens, two global permission guards reading disjoint metadata keys (47 vs 173 call sites, both executing per request, both inert when their decorator is absent), five Prisma services of which two share a class name, two `@Global` audit modules exporting the same class name, two policy engines, three idempotency implementations, two `AppException` classes in one directory, three auth/identity stacks.

E-03 retired one worry and confirmed another: the **order** of the nine global guards is safe (authentication precedes authorisation, downstream guards fail closed), while the **existence of two authorisation vocabularies** is confirmed and quantified. The structural problem is that correctness depends on which of several equivalent-looking subsystems a given module happened to import.

### 6.8 Theme F — Communication paths that carry nothing (ARCH-018)

The in-process event dispatcher has zero handler registrations, so every `dispatch()` returns after logging that no handlers are registered; a second outbox processor is constructed and never invoked; the analytics consumer is never provided and its queue is never registered. E-09 established that exactly one of six queues (`events`) has ever carried jobs. The consequence is structural: because the event paths do not carry, cross-module cooperation defaults to direct internal import — which is Theme D.

### 6.9 Theme G — Transaction boundaries (ARCH-019)

The transaction mechanism itself is well designed (ambient `AsyncLocalStorage` unit of work, a transaction pipe in the CQRS chain). The defect is that the guarantee is not universal: check-in and check-out post money-affecting SQL outside the command transaction, four services perform multi-step writes with no transaction, and raw SQL passes through the tenant proxy without joining the ambient transaction. Money- and inventory-affecting flows can therefore partially commit — a structural property of where boundaries are drawn, not a single faulty statement.

### 6.10 Theme H — Frontend structure (ARCH-021, ARCH-022, ARCH-023, ARCH-031)

Four structural defects with no backend coupling: four HTTP clients and five query-key registries with incompatible shapes for the same root key; three competing organizational schemes with a legacy⇄feature import cycle; 13 of 20 global stores fetching server state alongside React Query with no shared invalidation; and no boundary rules, with lint covering 259 of 871 web files. Because the frontend consumes the backend only over HTTP, these rows are structurally real and operationally contained.

### 6.11 Theme I — Verification structure (ARCH-024)

This row is architectural only in the sense that the system's *verification* structure is absent where the system's *behavioural* structure exists: 26 of 37 modules have no tests, there is no HTTP-level suite anywhere, api tests are excluded from typecheck, and — per E-10 — the database-gated suites are unset in every workflow and have no service container to connect to, while a passing spec asserts the presence of documentation text rather than the setting. It is the one architectural problem whose correction is purely additive: no relation is changed by adding coverage.

### 6.12 Theme J — Scaffolding and inert infrastructure (ARCH-027, ARCH-028, ARCH-032)

Three rows where the system presents capability it does not have: 19 of 37 modules are stub or thin and two are unregistered; a fully registered and annotated platform surface has zero first-party consumers and no reachable path (E-07 with E-04); and three Temporal layers exist around exactly one workflow that is never registered, with an empty workflow barrel and queues nothing listens on.

These are architectural — they concern what the system's shape claims versus what it delivers — but all three are dormant: they produce no incorrect behaviour today, which is why Document 01 judged them not currently justified to change and this document keeps that judgement.

### 6.13 What looks architectural but is not

| Item | Why it resembles an architectural problem | Final classification |
|---|---|---|
| ARCH-048 — 21 controllers on a doubled route prefix | Reshapes the API surface | `IMPLEMENTATION BUG` — the design (a global prefix) is correct; the implementation repeats it |
| ARCH-041 — legacy `availability` counter with database writers | Looks like split ownership of a counter | `LEGACY RESIDUE` — superseded machinery, dormant in traffic |
| ARCH-046 — stale inventory client | Looks like schema/client architecture drift | `IMPLEMENTATION BUG` — the schema and the database agree; the generated artifact is simply behind |
| ARCH-033 — unvalidated bodies, duplicated DTOs | Looks like an edge-architecture failure | `TECHNICAL DEBT` — the validation structure exists and is not used uniformly |
| ARCH-040 — api tests excluded from typecheck | Looks like a verification-architecture defect | `TECHNICAL DEBT` — a build configuration shortcut over an intact test structure |
| ARCH-036 — admin and mobile prototypes | Looks like portfolio/structure failure | `TECHNICAL DEBT` — two unfinished applications, structurally contained |

The distinction is not cosmetic: each of the six would be addressed in a different way from the twenty-six true structural defects, and conflating them is how a correction effort would misdirect itself.

---

## 7. Technical Debt

Eight rows remain `TECHNICAL DEBT`: recognised shortcuts and duplications that do not, by themselves, break the structure. None of them changed class in this step; E-07 and E-08 supplied evidence for two of them without altering their nature.

| Group | Rows | Position |
|---|---|---|
| **API edge quality** | ARCH-033 | 30 of 87 controllers accept `body: any`; two parallel reservation DTO sets are both consumed. The validation pipe exists; it is not applied uniformly. |
| **Frontend structure debt** | ARCH-034, ARCH-035 | Oversized components holding business rules; environment-dependent fallbacks and duplicated auth logic. Contained — no backend coupling. |
| **Unfinished applications** | ARCH-036 | E-08 split this into two distinct facts: admin is deployed with one API call site; mobile is connected with no deployment path at all. Different answers to "correction or preservation" follow from the two halves. |
| **Configuration and integration honesty** | ARCH-037, ARCH-038 | Two configuration roots and two feature-flag systems; stub integrations wired as if live, so operators cannot separate simulated from real behaviour. |
| **Documentation and build hygiene** | ARCH-039, ARCH-040 | Documentation describing capabilities the code does not have; api tests outside the type gate, with one test already importing a non-existent path. |

**Judgement:** none of these eight changes how the system must be built going forward, and none is a precondition for major-domain work. ARCH-038 and ARCH-040 come closest, because an operator who cannot tell live from simulated, and a test suite that compiles green while broken, both reduce the reliability of every observation made about the system — including observations used to plan corrections. They remain debt.

---

## 8. Legacy Residue

Six rows are `LEGACY RESIDUE`: superseded code or data that still exists and still affects behaviour. The step-02 evidence refined the *state* of this category rather than its size — one row was added from the resolved evidence (ARCH-052) and one existing row was re-characterised (ARCH-041).

| Row | State after evidence | Still affects behaviour? |
|---|---|---|
| **ARCH-041** (High) | **Re-characterised: dormant in traffic, live in DDL.** Two database-resident writers exist, are absent from every repository file, and fire on tables the application has abandoned (`allotment_pickup` = 0 rows, `allotment` = 1). The active pipeline writes other tables. | **Yes, if written to.** A single write to the legacy tables would mutate `availability` counters outside the assertion engine — and one database-gated spec performs exactly that write. |
| **ARCH-052** (Medium, absorbed) | **Resolved.** The mutators Document 01 could not identify are `trg_update_allotment_pickup` → `update_allotment_pickup()` and `trg_block_availability` → `block_availability_on_allotment()`, plus a timestamp trigger; `pg_cron` is absent, so no scheduled writer exists. | Same coupling as ARCH-041; absorbed into it and counted once. |
| **ARCH-043** (Medium) | Unchanged. The legacy `inventory_*` pipeline is written in production by purchasing, procurement, and housekeeping alongside `xylo_inventory`, with no bridge and no reconciliation. | **Yes — actively.** This is the residue that is *not* dormant. |
| **ARCH-044** (Medium) | **Reinforced.** E-05 enumerated 14 live tables with no Prisma model, including `folios`, `folio_postings`, `currencies`, `fx_rates`, `routing_instructions`, and `trx_code_config`. | **Yes.** Money tables are read and written through raw SQL outside the ORM's inventory of the database. |
| **ARCH-042** (Medium) | Unchanged. 82 files with zero non-test importers, including all three Reservations Phase-6 port adapters and seven handlers that would fail if routed. | **Yes, structurally** — dead code is indistinguishable from live wiring, which misleads every subsequent reading of the codebase. |
| **ARCH-045** (Low) | Unchanged. Tracked-but-deleted files, orphan SQL, tracked build artifacts, stale compiled specs. | Only to repository history and to anyone reading `git status` as a signal. |

**Judgement:** Document 01's conclusion — legacy is *active, not dormant* — is **confirmed for ARCH-043 and ARCH-044 and refined for ARCH-041/052**. The correct general statement is now: some residue is dormant in traffic but live in schema (availability triggers), some is fully active (legacy inventory writes), and some is misleading rather than executable (82 dead files). Any statement about "current behaviour" must still name which pipeline is meant.

Residue is not architecture: none of these six rows describes a boundary or ownership *relation* that is wrong by design. They describe state that outlived a decision. This is why none of them is a major-domain precondition, with the exception of ARCH-043, which is conditional because it shares its subject with ARCH-011.

---

## 9. Implementation Bugs

Five rows are `IMPLEMENTATION BUG`: correct design decisions, incorrectly implemented. The evidence step confirmed three of them and corrected a fact in a fourth.

| Row | Severity | Design that was correct | What the implementation did | Evidence |
|---|---|---|---|---|
| **ARCH-046** | **CRITICAL** | The inventory schema types `property_id` as `uuid`, matching the live database on all 17 columns | The installed `file:`-linked client was generated from `VarChar(20)` and is 4 models behind, and is imported at 20+ production sites | E-06, confirmed; "runtime errors today" remains `UNABLE TO VERIFY` |
| **ARCH-047** | **CRITICAL** | JWT signing keys belong in deployment configuration | The secret is inlined into the client bundle with hardcoded fallbacks | Document 01; untouched by this step |
| **ARCH-048** | MEDIUM | A single global route prefix | 21 controllers repeat the prefix in the path, producing `/api/v1/api/v1/...` | E-04, confirmed at framework-source level; no consumer uses either path form |
| **ARCH-049** | MEDIUM | A documented e2e verification command | The command points at a config file and directory that do not exist | Document 01; untouched |
| **ARCH-050** | MEDIUM | A worker disabled by configuration when its table is absent | The target table is absent, no migration drops it, and the disable flag is absent from the Kubernetes manifest — so the worker polls a nonexistent table on any deployment built from `infra/k8s` | E-05, confirmed with one fact corrected |

**Judgement:** two of the four Critical findings in the register are here, and both are in-place correctable with no structural consequence. They must not be conflated with the two Critical structural findings in Theme A. ARCH-048 is the clearest illustration of the classification boundary: the global-prefix design is right, the repetition of it is wrong, and the corrected class (`IMPLEMENTATION BUG`, not `ARCHITECTURAL PROBLEM`) is what keeps it out of the structural problem set.

None of the five is a major-domain precondition; ARCH-046 is conditional because its blast radius is the inventory domain.

---

## 10. Sound Architecture to Preserve

Seven rows are `SOUND` — working as intended, with evidence supporting the design. These are the properties a correction effort must not disturb.

| Row | Property that is working | Why it matters |
|---|---|---|
| **ARCH-001** | A coherent pnpm/Turborepo workspace with a single API composition root, and clean layer direction — `platform`, `common`, and `core` have **zero** edges into feature modules | Gives every later correction a stable composition point and an already-correct dependency direction |
| **ARCH-002** | Rebuilt domains implement genuine DDD: aggregates, value objects, domain events, domain errors, ports, repository interfaces and implementations across reservations, front-office, group-allotment, cashiering, availability | The reference pattern. It demonstrates the intended architecture is achievable *inside this codebase*, so structural correction is extension of a proven pattern rather than invention |
| **ARCH-003** | A custom CQRS bus with a five-pipe pipeline (validation → logging → authorization → transaction → idempotency), duplicate-registration guard, manual per-module registration, 188 handlers over 6 modules | A correct command boundary with built-in transaction and idempotency guarantees; the strongest argument that Theme E's duplication is fixable without losing capability |
| **ARCH-004** | A transactional availability assertion engine as single writer, with a disposable-schema Postgres harness that replays real migrations, guards against non-localhost, and gates 64+ specs | The model for database-integrated testing, and the only place where data correctness and its verification are provably paired |
| **ARCH-005** | Observability wired once and globally: OTel SDK init, a tracing interceptor, pino logging, request logging, exception filtering | A cross-cutting concern correctly placed at the composition root rather than per module |
| **ARCH-006** | Live OTA webhook ingress with HMAC-SHA256 verification, dedupe, and creation through the reservations facade | A correct inbound adapter pattern — the external boundary of the system is the healthiest part of the integration surface |
| **ARCH-053** | *(E-03)* Authentication precedes every authorisation enhancer, and the authorisation guards fail **closed** — an ordering regression would produce 403s, not bypasses | Newly established as sound. It retires Document 01's third open unknown in the safe direction and removes "ordering" from the list of things anyone needs to reason about |

Two further properties are recorded as sound in evidence but carry no finding row, because they were established during Step 02: the guard chain is fail-closed by construction, and exactly one of six queues has ever carried jobs with **nothing accumulating** on any of them.

---

## 11. Backend Repairability Assessment

### 11.1 Verdict

> **Backend repairability: `MODERATE — repairable with controlled migrations`.**
>
> **Backend preservation feasibility: High.**

The two are not in tension. *Repairability* describes how the defects are corrected — overwhelmingly through sequenced, verifiable changes that carry data, not through re-layout of the codebase. *Preservation feasibility* describes how much of the existing system survives that work: the HTTP contract, the composition root, the CQRS pipes, the observability stack, and the rebuilt domain cores all remain.

### 11.2 Distribution by repairability

| Class | Findings | Count |
|---|---|---:|
| **Repairable in place** — no structural relation changes; individually reversible | ARCH-024, ARCH-031, ARCH-042, ARCH-045, ARCH-046, ARCH-047, ARCH-048, ARCH-049, ARCH-050 | 9 |
| **Repairable with controlled migrations** — data-bearing, sequenced, verified per item | ARCH-007, **ARCH-008**, ARCH-009, ARCH-010, ARCH-011, ARCH-012, ARCH-013, ARCH-014, ARCH-015, ARCH-016, ARCH-017, ARCH-018, ARCH-019, ARCH-020, ARCH-021, ARCH-022, ARCH-023, ARCH-025, ARCH-026, ARCH-029, ARCH-030, ARCH-041, ARCH-043, ARCH-044, ARCH-051, ARCH-052 | 26 |
| **Major restructuring required (`LOW`)** | — | **0** |
| **Not currently justified to change** (no functional harm today) | ARCH-027, ARCH-028, ARCH-032, ARCH-036 | 4 |
| **Evidence required first** | — (all three resolved in Step 02) | **0** |
| Debt and residual rows not requiring structural classification | ARCH-033, ARCH-034, ARCH-035, ARCH-037, ARCH-038, ARCH-039, ARCH-040 | 7 |

Totals reconcile to 53: 9 in place + 26 controlled + 4 not justified + 7 debt rows = 46, plus 7 `SOUND` rows (ARCH-001…ARCH-006 and ARCH-053) which are outside the assessment.

Within the controlled-migration class, Document 01's high-risk subset is unchanged: **ARCH-007 and ARCH-008 (tenancy), ARCH-011 with ARCH-043 (dual inventory), and ARCH-012 (dual reservation/guest identity)** are data-bearing in a way where an incorrect order of operations corrupts tenant-visible data. That is recorded as a property of those rows, not as an order of work.

### 11.3 What the evidence changed about repairability

1. **The evidence-blocked class is empty.** Document 01 held ARCH-051, ARCH-052, and ARCH-053 outside any repairability class pending evidence. All three now carry a classification and a repairability position. No finding in the register is `NOT DETERMINABLE`.
2. **ARCH-010 became harder, in a specific way.** Document 01 treated it as repository hygiene. E-05 shows it is a *recovery* problem: the migration directories exist but are untracked, and the ledger has already diverged from the live database in two places. That is still a controlled-migration class problem — it is a matter of establishing a baseline that already exists on disk and in the database — but it is no longer a matter of committing files.
3. **ARCH-008 became larger and more precisely bounded.** 392 policies must be accounted for rather than "some RLS", and their inertness establishes that the current database's isolation cannot simply be assumed to be working. The class does not change; the size of the data-bearing work does.
4. **No finding moved into a worse class, and none moved into `LOW`.** There is no evidence anywhere in either input of an unresolvable tangle: one real module cycle, no frontend/backend source entanglement, and a proven in-repo reference pattern.

### 11.4 Why the verdict is `MODERATE` and not `HIGH`

`HIGH — repairable in place` would require that the structural defects be correctable without changing data. They are not. Twenty-six rows are data-bearing or relation-bearing: tenant scope, schema baseline, duplicated sources of truth, cross-module import depth, transaction boundaries. Each of those is correctable, and none requires re-architecture — but each carries data, and Document 01's own assessment recorded four as high-risk for exactly that reason. A verdict of `HIGH` would understate the sequencing and verification each of those requires; a verdict of `LOW` would be unsupported, since nothing in either input indicates major restructuring is needed anywhere.

### 11.5 Preservation feasibility detail

Preserved as-is: the HTTP surface and its `api/v1` prefix; explicit command routes in the rebuilt domains; the CQRS pipe chain; global observability; the cache subsystem with its distributed lock; the OTA ingress; the assertion engine and its test harness; the composition root and layer direction; and the DDD domain cores. Frontend corrections carry no backend risk at all, because there are zero production imports of the API or the Prisma package from the frontend.

---

## 12. Architecture Risk Concentration

### 12.1 Where the risk sits

| Concentration | Findings | Why it compounds |
|---|---|---|
| **Data layer** — tenancy, schema ownership, duplicated sources of truth, residue | ARCH-008, ARCH-009, ARCH-010, ARCH-011, ARCH-012, ARCH-020, ARCH-041, ARCH-043, ARCH-044, ARCH-051, ARCH-052 | **11 of 53 rows.** Every major domain writes here, and every correction in this group carries data — which is why it also holds the whole controlled-migration burden |
| **Seams between rebuilt domains** | ARCH-013, ARCH-014, ARCH-025, ARCH-026, ARCH-029, ARCH-030 | New work crosses these seams daily; front-office→reservations alone accounts for 35 of 78 cross-module internal-import edges |
| **Cross-cutting duplication** | ARCH-015, ARCH-016, ARCH-017, ARCH-018 | Every module must choose between two to five equivalents; choices diverge silently because the equivalents compile against the same tokens |
| **Verification** | ARCH-024, ARCH-040, ARCH-049 (+ E-10) | Refactors across the other concentrations are unguarded: 26 of 37 modules untested, no HTTP suite, api tests outside the type gate, DB suites skipped in CI |
| **Frontend structure** | ARCH-021, ARCH-022, ARCH-023, ARCH-031 | 612 of 871 web files are unlinted and boundaries are convention-only — contained, because frontend↔backend coupling is HTTP-only |
| **Security posture** | ARCH-007, ARCH-047 | One structural (tenant scope) and one implementation (secret exposure); they do not interact, but both are Critical |
| **Operational honesty** | ARCH-032, ARCH-038, ARCH-050, ARCH-018 | Provisioned-but-inert or simulated-but-wired subsystems make observations about the system unreliable — including observations used to plan corrections |

The rows overlap by design; each appears where it is most load-bearing.

### 12.2 Severity concentration

Four findings are `CRITICAL`, and they split cleanly:

- **ARCH-007 and ARCH-008 (structural, tenancy)** — global preconditions; every domain inherits them.
- **ARCH-046 and ARCH-047 (implementation)** — in-place correctable with no structural consequence; they are urgent, not compounding.

Seventeen findings are `HIGH`, of which sixteen are structural and one (ARCH-041) is legacy residue. The high-severity band is dominated by Themes A–D.

### 12.3 Where risk does **not** concentrate

- **The rebuilt domain cores.** ARCH-002, ARCH-003, and ARCH-004 are sound, and 88% of module tests sit in four of those domains.
- **Dependency direction.** `platform`, `common`, and `core` have zero edges into feature modules; there is one true reverse edge repo-wide.
- **Observability and inbound integration.** Wired globally once (ARCH-005); OTA ingress is live and verified (ARCH-006).
- **The queue backlog.** E-09: `wait = 0`, `active = 0`, `failed = 0` across all six queues — nothing is accumulating.
- **Frontend↔backend coupling.** Zero production imports of API source or Prisma, so frontend structure risk cannot propagate backward.

The overall shape of the risk statement: **risk is concentrated in the data layer and at the seams, is unguarded by verification, and is absent from the domain cores.**

---

## 13. Major-Domain Readiness Verdict

### 13.1 Scope and definitions

Assessed: the twelve substantive domains Document 01 §3.1 identified, **excluding Reservations**, which is outside this document's scope and is referenced below only where it is load-bearing for another domain's readiness.

| Verdict | Meaning |
|---|---|
| `READY` | No condition attaches |
| `READY WITH ARCHITECTURAL CONSTRAINTS` | The domain's own structure can receive new work, provided the global preconditions (§14, P1–P4) and the domain's named constraints hold |
| `NOT READY — ARCHITECTURAL CORRECTION REQUIRED FIRST` | The domain carries a defect in its own core object or is the primary vector of a global precondition; new work in it first requires that defect to be corrected |
| `NOT DETERMINABLE` | Evidence insufficient — **not used**; every domain below is determinable on the available evidence |

Because P1–P4 are global, **no domain receives an unconditional `READY`.** The differentiation below is over and above those four.

### 13.2 Per-domain verdicts

| Domain | Readiness verdict | Basis | Named constraints |
|---|---|---|---|
| **availability** | `READY WITH ARCHITECTURAL CONSTRAINTS` | Sound transactional core with its own DB-gated harness (ARCH-004); constraints are residue and wiring, not defects in the assertion design | ARCH-041/ARCH-052 (legacy counter with live database writers), ARCH-029 (mutual `forwardRef` cycle), six counter locations (Document 01 §6.3); P1–P4 |
| **activities** | `READY WITH ARCHITECTURAL CONSTRAINTS` | Bounded domain with real tables, but its behaviour lives in controllers over raw SQL | ARCH-014 (controller-owned SQL); further behaviour there deepens the layering defect; 7 specs; P1–P4 |
| **cashiering** | `READY WITH ARCHITECTURAL CONSTRAINTS` | Domain services and 23 CQRS registrations exist | ARCH-014 (handlers reach Prisma directly), ARCH-019 (money writes outside transaction boundaries), ARCH-044 (folio tables outside the ORM); P1–P4, P8 |
| **channels** | `READY WITH ARCHITECTURAL CONSTRAINTS` | Inbound side is the healthiest integration surface in the system (ARCH-006) | ARCH-038 (outbound side simulated, metrics fabricated); 1 spec; P1–P4 |
| **command-center** | `READY WITH ARCHITECTURAL CONSTRAINTS` | Own aggregates and widget registry; only domain on the second CQRS stack | ARCH-015 (second bus with identical tokens — a third idiom must not appear); 4 specs; P1–P4 |
| **front-office** | **`NOT READY — ARCHITECTURAL CORRECTION REQUIRED FIRST`** | It is the primary vector of the cross-domain precondition: 35 internal-import edges reaching seven layers of another domain, plus money-affecting writes outside its own transaction and direct writes into folio data | ARCH-013 (global precondition P4), ARCH-019 (P8), ARCH-025 (P10); 23 specs exist and do not offset the boundary defect |
| **group-allotment** | `READY WITH ARCHITECTURAL CONSTRAINTS` | Full DDD and the densest test coverage in the repository (75 specs); its own controllers reach Prisma directly | ARCH-014; ARCH-041/ARCH-052 — one database-gated spec writes the legacy trigger tables; P1–P4, P8 |
| **housekeeping** | **`NOT READY — ARCHITECTURAL CORRECTION REQUIRED FIRST`** | Its core object does not exist: `person_discrepancies` is referenced by 15 raw-SQL statements in production code while being absent from the live database and from all 51 migrations (Document 02 §9, recorded at Document 02 §16.2.10 as evidence, **not** as a new finding ID) | Runtime `CREATE TABLE` at boot (ARCH-020); whether those code paths execute in production is an open residual (§16, R9) and the verdict holds either way |
| **inventory** | **`NOT READY — ARCHITECTURAL CORRECTION REQUIRED FIRST`** | The client the API compiles against diverges from both the schema and the live database (`uuid` vs `VarChar(20)`) and is four models behind, at 20+ import sites; the domain also sits on one half of a dual pipeline | ARCH-046 (P5), ARCH-011 (P6), ARCH-043; **0 tests across 282 files**; P1–P4 |
| **purchasing** | **`NOT READY — ARCHITECTURAL CORRECTION REQUIRED FIRST`** | A single 2,777-line service with 146 Prisma call sites operating on the legacy inventory world, with no tests | ARCH-011, ARCH-043 (P6); P1–P4 |
| **billing** | **`NOT READY — ARCHITECTURAL CORRECTION REQUIRED FIRST`** | A three-file module whose data is written from another module, whose only payment-gateway consumer is a stub, and which has no tests | ARCH-025 (P10), ARCH-038; 0 specs; P1–P4 |
| **rates-inventory** | `READY WITH ARCHITECTURAL CONSTRAINTS` | Six domain services and partial DDD; rate engine is a real bounded capability | ARCH-029 (cycle with availability); rests on the dual reservation/guest models (ARCH-012, P9); 5 specs; P1–P4 |

### 13.3 Overall readiness verdict

> **No major domain is unconditionally `READY`. Seven are `READY WITH ARCHITECTURAL CONSTRAINTS`. Five are `NOT READY — ARCHITECTURAL CORRECTION REQUIRED FIRST`. Zero are `NOT DETERMINABLE`.**

The five share a pattern: each has a defect **in its own core object** — an absent table (housekeeping), a stale compiled client (inventory), data owned elsewhere (billing), a single-class god service on the superseded pipeline (purchasing) — or it is the primary vector of the global cross-domain precondition (front-office). None of the five is blocked by something outside its boundary except front-office, and that is precisely why front-office and ARCH-013 are linked in §14.

The seven constrained domains are constrained by the *same four* global conditions plus one or two local ones. Their structure — modules, repositories, handlers, tests — can receive work.

---

## 14. Architectural Preconditions

These are **conditions that must hold**, expressed as states of the system. They are not work items, they carry no order, no owner, no sequence, and no estimate. Document 01 named three preconditions (ARCH-007, ARCH-008, ARCH-013); this document adds one (ARCH-010) on the strength of Document 02 §9, and records seven scoped conditions.

| Ref | Condition that must hold | Scope | Source finding(s) |
|---|---|---|---|
| **P1** | Tenant and property scope for every non-public request is established from authenticated identity; no client-supplied header is authoritative for it | **Global** | ARCH-007 (+ Document 02 §16.5) |
| **P2** | The row-level isolation policy that scopes tenant data is declared in this repository and holds for the database role the application connects as — or its absence is recorded as an explicit, reviewed property of the deployment | **Global** | ARCH-008, ARCH-051 |
| **P3** | The schema and migration baseline for all three Prisma schemas is recoverable from version control, and the migration ledger, the directories on disk, and the live database agree | **Global** | ARCH-010, ARCH-009 |
| **P4** | Cross-domain use of the reservations domain occurs through the surface that domain exports; no import reaches its internal layers | **Global** | ARCH-013 |
| **P5** | The inventory Prisma client in use reflects the current inventory schema and the live column types | Inventory | ARCH-046 |
| **P6** | Each purchasing, stock, and warehouse concept resolves to a single authoritative store | Inventory, purchasing | ARCH-011, ARCH-043 |
| **P7** | Every mutating endpoint carries an authorization requirement with unambiguous semantics, and an endpoint without one is not reachable | Any new API surface | ARCH-016 |
| **P8** | Every multi-step write that affects money or inventory completes within a single transaction boundary | Money- and inventory-affecting paths | ARCH-019, ARCH-025 |
| **P9** | Guest identity data resolves to a single authoritative record for any work that reads or writes guest contact or identity fields | Guest-identity-facing work | ARCH-012 |
| **P10** | Folio and charge data has a single writer, and that writer is the module that owns the folio | Folio/billing work | ARCH-025 |
| **P11** | A module whose behaviour changes carries executable evidence of the behaviour it changed | The 26 modules with no tests | ARCH-024, ARCH-040 (+ E-10) |

**Not a domain precondition:** ARCH-047 (JWT signing secret in the client bundle) is recorded here so it is not lost behind the structural list. It conditions deployment safety, not domain work, and it is in-place correctable.

P1–P4 are the four `YES` rows of the register. P5–P11 correspond to the eight `CONDITIONAL` rows, each scoped to the work it conditions.

---

## 15. Overall Architecture Verdict

> ## STRUCTURALLY PROBLEMATIC BUT REPAIRABLE — confirmed, sharpened, and now fully evidenced.

Document 01's headline stands. Three amendments follow from the evidence, and none of them softens it.

**1. The problem set is closed.** All 53 rows now carry a classification, a severity, and evidence; the register contains **zero** `UNCERTAIN` rows where it previously held three. The shape of the problem is known rather than estimated — including in the one area where Document 01 explicitly said it could not tell (database isolation).

**2. The precondition set grows from three findings to four.** ARCH-007, ARCH-008, and ARCH-013 remain as Document 01 stated. **ARCH-010 joins them**: version control does not contain the schema history the live database has applied, the ledger has diverged from the database in two known places, and every major domain writes schema. Document 02 quantified this in a way Document 01 could not (51 directories, 23 tracked, 28 untracked, 2 unfinished ledger rows, 1 table dropped outside history, 1 table referenced 15 times and never created).

**3. The tenancy defect is larger and differently urgent than Document 01 recorded — and its Critical rating stands for a sharper reason.** Document 01 could say only that isolation was "not reproducible from source". The evidence shows 386 RLS-enabled tables and 392 policies that the repository does not reproduce, *and* that they are inert for the role that connects (owner, `SUPERUSER`, `BYPASSRLS`, no `FORCE`, and a session setting that is never set and is named differently by its setters than by the policies). The consequence is symmetric and it is the decisive architectural fact of this investigation:

> **In the live database, tenant isolation exists but does not enforce. In any environment this repository can produce, it does not exist. There is therefore no configuration in which database-level tenant isolation currently holds — and the only boundary that does operate is a client-controllable header.**

That is a sharper statement than "unreproducible", and it is why ARCH-007 and ARCH-008 remain the two `CRITICAL` structural findings.

**Supporting judgements:**

- **Repairability: `MODERATE — repairable with controlled migrations`** (§11). No finding requires major restructuring; no finding remains evidence-blocked; 26 rows are data-bearing and therefore sequenced rather than in-place.
- **Preservation feasibility: High.** The HTTP contract, composition root, CQRS pipes, observability stack, OTA ingress, assertion engine, and DDD domain cores are all preserved. Nothing in either input indicates the backend needs rebuilding.
- **The centre holds; the seams and the data layer do not.** Sound domain cores, clean dependency direction, and a working reference pattern sit inside a data layer with duplicated ownership, a version-control gap over its history, and unenforced module boundaries.
- **Frontend risk is real and contained** — four structural defects, zero backend coupling.
- **Verification is the amplifier.** Every other concentration is harder to correct safely because 26 of 37 modules have no tests, there is no HTTP suite, and the database-gated suites never run in CI.
- **Two urgent defects are cheap.** ARCH-046 and ARCH-047 are `CRITICAL` and in-place correctable; they are deliberately excluded from the structural precondition list so they are not treated as equivalent to it.

**What would falsify this verdict:** evidence that tenant scope is enforced by some mechanism not examined in either input; or a demonstrated, reproducible schema baseline outside version control; or an unresolvable dependency tangle. Document 01 searched for the first two and Document 02 tested them directly; neither was found.

---

## 16. Open Residual Uncertainties

Nine items remain open after Step 02. Each is listed with why it is open and what its effect on this document's verdicts would be.

| Ref | Open item | Why it remains open | Effect on the verdicts in §11, §13, §15 |
|---|---|---|---|
| **R1** | Is the stale inventory client producing runtime errors today? (E-06 sub-question) | API not running during Step 02; no retained application logs | **None.** ARCH-046 is already P5 on type-level evidence; runtime symptoms would not change its class |
| **R2** | Are there external, non-repository callers of the platform surface? (E-07 sub-question) | No access log, HAR, or capture exists; API not running | **None.** ARCH-028 is confirmed for all first-party consumers; an external caller would not make the surface consumed in any architectural sense |
| **R3** | Runtime capture of the enhancer array (E-03 sub-question) | Booting the API starts pollers and publishers against the shared live database | **None.** The order is derived from the exact installed framework source and the guards are fail-closed regardless of order |
| **R4** | A single live HTTP round-trip confirming `/api/v1/api/v1/...` (E-04 sub-question) | Same — API not booted | **None.** The concatenation rule is unconditional in the installed library and no consumer uses either path form |
| **R5** | Column-level diff of all 1,018 models against the live database (E-05 sub-question) | `prisma db pull` rewrites `schema.prisma`; `prisma migrate diff` writes a shadow database | **None.** ARCH-010 is already a global precondition at table and ledger level; column detail could add specificity, not change class |
| **R6** | Which actor updated `availability` on 2026-09-17 (E-02 sub-question) | No audit trail on that table beyond `updated_at`; API not running | **None.** ARCH-041/ARCH-052 rest on object provenance and table activity, not on that single timestamp |
| **R7** | Live rather than persisted queue census (E-09 sub-question) | API not running; census is last-run persisted state | **None.** The wiring matrix is static and complete; the operational claim made here (`wait = 0` everywhere) is explicitly a persisted-state claim |
| **R8** | Provenance of the `bull:analytics:meta` key (Document 02 §16.2.14) | The queue is not registered in `queue.module.ts` yet has a Redis key; origin not attributable from source | **None.** It does not alter the queue-wiring conclusions that support ARCH-018's refinement |
| **R9** | Whether housekeeping's `person_discrepancies` code paths execute in production (Document 02 §16.2.10) | Table absence is proven; execution frequency is not observable without running the API | **None — the verdict is robust to both outcomes.** If the paths execute, they error; if they never execute, the domain's core feature is inert. Both support `NOT READY` |

**No open residual uncertainty changes the repairability verdict (§11), any readiness verdict (§13), or the overall verdict (§15).** Every item above would add specificity to an existing row or basis; none would reclassify one.

---

## 17. Explicit Non-Decisions

The following were deliberately **not** decided in this step:

1. **No target architecture.** No future-state design, component diagram, or end-state description is produced here. That is a later step's scope.
2. **No gap analysis.** The distance between current and target state is not measured, because no target state exists yet.
3. **No correction plan.** No remediation sequence, wave, phase, owner, task, effort estimate, dependency graph, or workstream appears in this document. §14 states conditions only and implies no order.
4. **No Reservations decision.** The Reservations rebuild, its phases, its specs, and its readiness are outside this document's scope. Reservations appears only where it is load-bearing for another domain's readiness (front-office, ARCH-013/P4) or as evidence within an existing row.
5. **No decision on which authorization mechanism survives.** ARCH-016 and P7 record that a single, unambiguous authorization requirement must hold; which of the two existing mechanisms is the one that holds it is not chosen here.
6. **No decision on schema unification.** ARCH-009, ARCH-011, ARCH-012, and P6 record that ownership is duplicated and that a single authoritative store is the condition; whether the three schemas are unified, and how any concept's store is arrived at, is not decided here.
7. **No decision on unfinished surfaces.** ARCH-027 (stub and thin modules), ARCH-028 (unconsumed platform surface), ARCH-032 (inert Temporal), and ARCH-036 (admin/mobile) are recorded as not currently justified to change. Their eventual disposition is not decided here.
8. **No decision on communication paths.** ARCH-018 records that in-process event dispatch does not carry and that one of six queues has ever carried jobs; which mechanism the system uses for cross-module cooperation going forward is not decided here.
9. **No frontend target structure.** ARCH-021/022/023/031 record the four structural defects; the organizational scheme, client count, state strategy, and lint policy that follow are not decided here.
10. **No test strategy.** ARCH-024 and P11 record that behavioural change in untested modules requires executable evidence; coverage targets, frameworks, and suite organisation are not decided here.
11. **No runtime verification was performed in this step.** The API was not booted, no HTTP request was issued, no queue was produced to, and no database statement other than read-only catalogue selection was executed at any point in Steps 01–03.
12. **No modification of Documents 01 or 02.** Their findings, numbering, and recorded classifications are untouched; this document supersedes nothing and adds no finding IDs.

---

*End of Document 03. No source code, database schema, migration, API, business logic, CI configuration, deployment manifest, or frontend behaviour was modified in the course of this step. Documents 01 and 02 were read in full and left unaltered. No architecture documents other than this one were created, and no finding identifier outside ARCH-001…ARCH-053 was introduced.*



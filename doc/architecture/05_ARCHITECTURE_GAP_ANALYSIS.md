# Document 05 — Architecture Gap Analysis

**Step 05 of the XYLO Project-Wide Architecture Review**

Status: COMPLETE — ANALYSIS ONLY
Follows: `docs/architecture/01_CURRENT_ARCHITECTURE_AUDIT.md`, `docs/architecture/02_OPEN_EVIDENCE_RESOLUTION.md`, `docs/architecture/03_ARCHITECTURE_FINDINGS_AND_VERDICT.md`, `docs/architecture/04_TARGET_ARCHITECTURE.md`
Date: 2026-10-08

---

## 1. Purpose & Scope

This document identifies and classifies the gaps between the **confirmed current state** (Documents 01–03) and the **locked target architecture** (Document 04). Its single product is the Gap Register (§4).

It does **not**: modify code, schemas, migrations, the database, tests, or CI/CD; refactor anything; implement fixes; start Reservations; create an implementation plan or remediation sequence; resolve any Document 04 Open Decision; reopen Documents 01–04; or create new current-state findings. No new empirical evidence was gathered in this step — the analysis reads only the four authoritative documents.

Multiple findings from Document 03 are consolidated: **53 findings resolve into 26 register entries**, of which 22 are confirmed architectural gaps. One finding is not automatically one gap.

## 2. Inputs / Authority

| Input | Role in this analysis |
|---|---|
| `01_CURRENT_ARCHITECTURE_AUDIT.md` | Current-state facts, evidence locations, finding register (ARCH-001…ARCH-053) |
| `02_OPEN_EVIDENCE_RESOLUTION.md` | Resolved evidence items E-01…E-10 used as current-state proof |
| `03_ARCHITECTURE_FINDINGS_AND_VERDICT.md` | **Authoritative current-state classification, severities, preconditions P1–P11, readiness verdicts, residuals R1–R9** |
| `04_TARGET_ARCHITECTURE.md` | **Authoritative target rules**: principles §1, boundaries §2–§4, constraints AC-01…AC-43 (§15), open decisions OD-01…OD-10 (Appendix A) |

Documents 01–04 are unchanged by this step. Where this document cites a finding ID or an evidence ID, it cites it as *basis*; it creates no `ARCH-` identifier and no new current-state claim.

## 3. Gap Classification Rules

### 3.1 What can be a gap

| Kind | Treatment |
|---|---|
| **Architectural gap** — current implementation violates a Document 04 rule | Counted |
| **Technical debt** | Counted **only** where Document 04 explicitly requires a different state (e.g. untyped request bodies → AC-26). Otherwise not a gap |
| **Implementation bug** | Counted **only** where it violates a target constraint (e.g. ARCH-046 violates AC-21). Purely behavioural bugs are out of scope |
| **Open Decision dependency** | Not an architectural failure by itself; either noted in the OD column of a confirmed gap, or classified `CONDITIONED BY OPEN DECISION` |

### 3.2 Classification values

| Value | Meaning |
|---|---|
| `CONFIRMED` | The violation is provable now from Documents 01–03 against a Document 04 rule that holds regardless of any open decision |
| `CONDITIONED BY OPEN DECISION` | Whether the rule applies — or in what form — depends on an OD from Document 04 Appendix A; no violation is asserted until the OD closes |
| `NOT A GAP` | Current state differs from the target's vocabulary, but the target does not prohibit it — or the target explicitly preserves the current property |
| `DEFERRED` | Outside this step's analytical scope, or awaiting a decision/evidence class this step does not resolve |

### 3.3 Identifiers and severity

- **`GAP-nn`** — register entries in this document. Analysis artefacts, **not** findings; neither `GAP-` nor `N-` identifiers are finding identifiers, and no new `ARCH-` identifiers exist.
- **`N-nn`** — non-gap / deferred entries (§8).
- **Severity** — `CRITICAL / HIGH / MEDIUM / LOW`, describing the gap's architectural impact, informed by (but not identical to) the Document 03 severity of its supporting findings. Severity is not a work order and implies no sequence.

### 3.4 What was deliberately not done

No gap is manufactured to increase counts; no gap is split where findings share one architectural cause; no remediation, ordering, ownership, or estimate appears anywhere in this document.

---

## 4. Gap Register

The register is the primary artefact. Every row states the gap, its current-state evidence, the target rule it violates, severity, affected domains, open-decision dependency, and classification.

| ID | Gap | Current-state evidence | Target rule (Doc 04) | Severity | Affected domains | OD | Classification |
|---|---|---|---|---|---|---|---|
| GAP-01 | Tenant/property scope originates from a client-controllable header with property authorization disabled, rather than from the authenticated principal. | ARCH-007 (CRITICAL); Document 03 P1 | AC-01; §1.2; §7.1 | CRITICAL | All contexts (Identity & Access boundary) | OD-01 | CONFIRMED |
| GAP-02 | No effective database-level isolation exists in any environment this repository can produce: 386 policies over 386+ tables exist only live, under an owner+superuser+bypass role, with zero forced enforcement and setters writing a scope parameter readers never read. | ARCH-008 (+ ARCH-051 absorbed); Document 02 E-01; Document 03 P2 | AC-03; AC-04; §1.4; §7.4–7.5 | CRITICAL | All contexts (data layer) | OD-02 | CONFIRMED |
| GAP-03 | Authorization is not single and uniform: two disjoint permission vocabularies (47 and 173 sites) both evaluate on every request, and both go inert when their decorator is absent. | ARCH-016 (residual of ARCH-053); Document 02 E-03; Document 03 P7 | AC-23; AC-24; §1.5; §9.6 | HIGH | All API surfaces | OD-09 | CONFIRMED |
| GAP-04 | Contexts depend on each other outside contracts: 78 cross-module internal-import edges (front-office into reservations' internals across 7 layers on 35 edges), a reverse infrastructure→modules edge, and a mutual Availability⇄Rates synchronous cycle. | ARCH-013; ARCH-030; ARCH-029; Document 03 P4 | AC-06; AC-07; §1.3; §4.3; §4.4; §5.1 | HIGH | front-office, reservations, availability, rates-inventory, all backend | — | CONFIRMED |
| GAP-05 | Persistence is reached outside the owning context: 233 module files use `PrismaService`, 144 use raw SQL, 6 controllers inject it directly, the shared repository layer has 2 references, and `common/audit` writes into the inventory schema (`inv_audit_log`). | ARCH-014; Document 01 (cross-schema audit write) | AC-14; §4.3.2–4.3.3; §5.1.3; §6.2 | HIGH | All backend (activities, cashiering, platform most exposed) | — | CONFIRMED |
| GAP-06 | The schema/migration baseline is unreproducible: ledger, directories, and database disagree (28 of 51 migrations untracked, 2 unfinished ledger rows, one table dropped outside history, one referenced 15× and never created), `platform.prisma` is untracked, and 14 live tables have no model. | ARCH-010; ARCH-044; ARCH-045; ARCH-050; Document 01 §6.1; Document 02 E-05; Document 03 P3 | AC-18; §8.1; §8.3 | HIGH | All (database); billing (folio tables unmodelled) | — | CONFIRMED |
| GAP-07 | Schema and database-resident behaviour change outside migrations: two services execute `CREATE TABLE`/`CREATE INDEX` at boot; 386 policies and two data-mutating triggers exist with no source in any migration. | ARCH-020; ARCH-008; ARCH-041; ARCH-052; Document 02 E-01, E-02 | AC-19; AC-20; §1.4; §6.8; §8.5 | HIGH | All; availability (legacy triggers) | — | CONFIRMED |
| GAP-08 | The generated client in use is not reproducible from committed schema: a `file:`-linked inventory client declares `VarChar(20)` against live/schema `uuid`, carries 4 ghost models, and is imported at 20+ production sites. | ARCH-046 (CRITICAL); ARCH-011; Document 02 E-06; Document 03 P5 | AC-21; §8.4 | CRITICAL | inventory, purchasing | — | CONFIRMED |
| GAP-09 | Data-isolation granularity is below the target minimum (907 models for most contexts share one schema beside `xylo_inventory` and `xylo_platform`, over one physical database) and cross-schema keys are untyped strings. | ARCH-009; Document 01 §6.1 | §8.8.1; §6.7.2 | MEDIUM | All (database) | OD-07 | CONFIRMED |
| GAP-10 | The one-mechanism rule is violated across cross-cutting concerns: two CQRS buses with identical tokens, two audit systems, two policy engines, three idempotency paths, two error types, four HTTP client stacks, and two configuration roots with two feature-flag systems. | ARCH-015; ARCH-017; ARCH-021; ARCH-037 | §1.6; AC-09 | HIGH | All; command-center (second bus); web/admin (clients, config) | — | CONFIRMED |
| GAP-11 | Communication and job mechanisms exist in parallel and carry nothing: an in-process dispatcher with zero registrations, a second outbox processor with no callers, an unregistered analytics consumer, three overlapping Temporal layers whose single workflow was never registered, and queues nothing listens on — only the `events` queue carries traffic. | ARCH-018; ARCH-032; Document 02 E-09 | AC-36; AC-38; §5.3.6; §5.6; §11.5 | HIGH | All; platform (workflow runtime) | OD-03 | CONFIRMED |
| GAP-12 | Inert or unconsumed subsystems are neither active nor declared dormant: 19 of 37 modules stub/thin with 2 unregistered, a platform surface with 173 permission annotations and zero first-party consumers, and 82 files with zero non-test importers. | ARCH-027; ARCH-028 (+ E-07); ARCH-042 | §1.9; §2.7.3; §13.6; §14.6 | MEDIUM | platform surface; inventory (empty structure); backend modules | — | CONFIRMED |
| GAP-13 | Public routes resolve at a doubled prefix (`/api/v1/api/v1/…`) on 21 controllers, and the Swagger contract advertises the doubled path while no consumer uses either form. | ARCH-048 (bug-class); Document 02 E-04; Document 01 (swagger setup) | AC-27; §9.1.3 | MEDIUM | platform/API; web consumers | — | CONFIRMED |
| GAP-14 | Contract types are duplicated and request bodies untyped: two parallel reservation DTO sets are both consumed, duplicate DTOs exist for property/company, and 30 of 87 controllers accept `@Body() body: any`. | ARCH-033; ARCH-026 (DTO aspect) | AC-26; §9.4 | MEDIUM | API surface; property/company | — | CONFIRMED |
| GAP-15 | The target's verification contract is unmet: DB-gated suites skip on every PR (test DB URL absent from CI), 26 of 37 modules are untested, tests are excluded from typecheck, no boundary/architectural checks exist, api lint is `echo 'ok'`, and the documented e2e command points at files that do not exist. | ARCH-024; ARCH-040; ARCH-039; ARCH-049; ARCH-031; Document 02 E-10; Document 03 P11 | AC-40; AC-41; §1.10; §13.3–13.5 | HIGH | All; web (lint covers 209 of 871 files) | OD-10 | CONFIRMED |
| GAP-16 | Legacy paths violate single-writer and declaration rules: `public.inventory_*` is still written by purchasing, procurement, housekeeping, and stock/receiving alongside `xylo_inventory` with no bridge, and data-mutating triggers sit on abandoned tables, absent from every repository file. | ARCH-043; ARCH-041; ARCH-052; Document 01 (two inventory worlds); Document 02 E-02; Document 03 P6 | AC-42; AC-43; §1.13; §6.6; §14.2–14.4 | HIGH | inventory, purchasing, housekeeping, availability | — | CONFIRMED |
| GAP-17 | Money-affecting multi-step writes occur outside a single transaction boundary: check-in/check-out post money-affecting SQL outside the command transaction, and four services perform multi-step writes with no transaction at all. | ARCH-019; Document 03 P8 | AC-13; §1.7; §6.3 | HIGH | front-office, cashiering, billing; four unidentified services | — | CONFIRMED |
| GAP-18 | Several concepts have two or three live authoritative stores: dual inventory worlds, dual reservation and triple guest sources of truth, two owners for property and company, folio data written from outside Billing, and six-plus capacity counter locations. | ARCH-011; ARCH-012; ARCH-026; ARCH-025; Document 01 §6.3; Document 03 P6, P9, P10 | AC-12; §1.1; §3.1; §3.3 | HIGH | inventory, purchasing, guest-profile, reservations, billing, property/company, availability | — | CONFIRMED |
| GAP-19 | The web/admin frontend has no single server-state layer and no enforced feature boundaries: four HTTP clients, five incompatible query-key registries, 13 of 20 stores fetching server state, double-cached endpoints with no shared invalidation, three independent `Reservation` types, a legacy↔feature import cycle with 19 `app/` files reaching into features, and nine hardcoded host fallbacks. | ARCH-021; ARCH-022; ARCH-023; ARCH-035 (hosts) | AC-30; AC-31; AC-32; §10.2–10.4 | HIGH | web, admin | — | CONFIRMED |
| GAP-20 | Client auth integrity is void: the JWT signing secret is inlined into the client bundle with hardcoded fallbacks, auth logic is duplicated across web and admin, and cookie access exists in three copies. | ARCH-047 (CRITICAL); ARCH-035 (auth, cookie) | AC-33; §10.6 | CRITICAL | web (admin: duplicated handling) | — | CONFIRMED |
| GAP-21 | Outbound integration behaviour is simulated or fabricated without labelling: payment gateway stub hardwired, push SDKs absent, channel metrics fabricated — operators cannot separate live from simulated. | ARCH-038 | AC-35; §1.9; §11.3 | HIGH | channels, billing | — | CONFIRMED |
| GAP-22 | Business rules are held outside domain layers: rules live inside 46 oversized UI components, and activities' behaviour lives in controllers over raw SQL. | ARCH-034 (rules-in-UI); ARCH-014 / Document 03 §13.2 (activities controllers) | §1.8; §4.1 (presentation row) | MEDIUM | web, activities | — | CONFIRMED |
| GAP-23 | Reporting/analytics consumes authoritative state through trigger-coupled ETL (`onFolioPost`, `onNightAuditComplete`) whose only consumer is dead; the target path (read contracts vs rebuildable projections) is undecided. | Document 01 (analytics ETL via reporting-analytics; dead `AnalyticsConsumer`); ARCH-018 | §6.5; §2.6 (OD-05) | MEDIUM | reporting, billing, cashiering | OD-05 | CONDITIONED BY OPEN DECISION |
| GAP-24 | Tenancy naming is not unified across one isolation root: `app.current_tenant` vs `app.hotel_id` scope parameters, and `hotel_id` vs `property_id` keys on same-grain tables. | ARCH-008 (parameter mismatch); Document 01 §6.1 (`channel_availability` vs `channel_inventory`) | AC-02 (OD-01-dependent per §15.10); §7.7 | MEDIUM | All (database, contracts) | OD-01 | CONDITIONED BY OPEN DECISION |
| GAP-25 | A single machine-readable contract per surface — public, internal, integration — is not demonstrable: Swagger exists but its coverage of internal/integration surfaces is unconfirmed and it advertises the doubled path. | Document 01 (swagger setup); Document 02 E-04, E-07 | AC-25 (OD-04-dependent per §15.10); §9.5 | MEDIUM | platform/API | OD-04 | CONDITIONED BY OPEN DECISION |
| GAP-26 | Cross-domain identifier format is inconsistent (UUID, text, composite primary keys mixed); the target convention is undecided. | ARCH-009 (mixed key forms; untyped cross-schema keys) | §6.7.3 (OD-08); typing itself is GAP-09 | LOW | All (contracts, persistence) | OD-08 | CONDITIONED BY OPEN DECISION |

---

## 5. Global / Cross-Cutting Gaps

These gaps are global because their target rules are global (Document 03 P1–P4 class) or because the violated constraint applies to every context.

**GAP-01 and GAP-02 are the two halves of the isolation contract.** GAP-01 is the application layer: scope is selected by the client instead of derived from identity, so there is no boundary that operates on the request path except one that the client influences. GAP-02 is the database layer: the defence-in-depth that would catch GAP-01 does not exist in any environment the repository can produce. Together they mean neither of the two required enforcement layers (§7.4) currently functions. GAP-01's violation is independent of OD-01; GAP-02's violation is independent of OD-02 — the open decisions select the *shape* of the correct state, not whether the current state violates it.

**GAP-03** sits on the request path itself: the target requires one mechanism with unambiguous semantics evaluated uniformly (AC-23/AC-24); today two disjoint vocabularies coexist and an absent decorator means no check at all. OD-09 decides which vocabulary survives; the uniformity violation exists now.

**GAP-04 and GAP-05** are the boundary pair: GAP-04 is *code-level* dependence on other contexts' internals (imports, reverse edges, cycles), GAP-05 is *data-level* access reaching past owners (ORM clients, raw SQL, controllers, cross-schema writes). GAP-04 subsumes Document 03 P4's subject matter; GAP-05 shows why contracts are the only safe surface. Both are provable independently of any open decision.

**GAP-06, GAP-07, GAP-08, GAP-09** form the persistence/schema group and ladder from reproducibility (06) to change channel (07) to client trustworthiness (08) to isolation granularity (09). GAP-06 and GAP-07 are Document 03 P3 territory; GAP-08 is P5's type-level proof. GAP-09 is confirmed on two unconditional rules — per-context schema minimum and typed keys — while OD-07 governs only whether any context escalates beyond that minimum.

**GAP-10 and GAP-11** are the one-mechanism violations: GAP-10 covers concerns that must have exactly one *reachable* implementation (§1.6 — including configuration); GAP-11 covers the communication/job family where parallel stacks exist and none demonstrably carries facts (AC-36/AC-38). OD-03 selects which mechanism family wins; it does not make five parallel carriers compliant.

**GAP-12** is the capability-honesty gap: the target permits a subsystem to be active or declared dormant, never silently inert (§1.9, §2.7.3). **GAP-13 and GAP-14** are the API-surface gaps: the route prefix defect and contract-type duplication/untyped bodies both violate rules that hold regardless of how contracts are eventually published (OD-04).

**GAP-15** is structural, not aspirational: the target treats machine-enforced verification as part of the architecture (§1.10) and makes silent skipping a failure (§13.3). Today no boundary check exists anywhere, gated suites skip on every pull request, and tests sit outside the type gate — so no target rule in this register is currently enforced by any gate.

**GAP-16** is the legacy group where the target explicitly requires a different state: exactly one authoritative pipeline per concept and declared legacy status (AC-42/AC-43). Legacy *existing* is not the gap; legacy being live, undeclared, and written alongside the current pipeline is.

Interlock note: GAP-15 (no machine enforcement) means every other gap in this register is currently protected only by convention — which §1.10 classifies as "not yet an architectural rule".

## 6. Domain-Specific Gaps

**GAP-17 (transaction boundaries).** Document 03 P8's subject: money and inventory operations commit partially or outside any transaction (ARCH-019). The target's rule is unconditional (§1.7, AC-13); no open decision touches it.

**GAP-18 (multiple authoritative stores).** The single-owner principle (§1.1) is the target's first principle, and five concept families currently have two or three live writers. This gap consolidates ARCH-011, ARCH-012, ARCH-025, ARCH-026 and Document 01's counter-location evidence — one architectural cause (no single owner enforced), five expressions. Which store *survives* per concept is Document 03 P6/P9/P10 territory, not this document's.

**GAP-19 and GAP-20 (frontend).** GAP-19 is structural: one server-state layer, contract-sourced types, machine-enforced feature boundaries (AC-30…AC-32) — today four client stacks, five cache-key shapes, three `Reservation` types, and lint covering under a quarter of the web codebase. GAP-20 is the client-trust violation: a signing secret in the bundle voids token integrity outright (AC-33).

**GAP-21 (integration outbound).** Inbound integration is sound (ARCH-006, preserved per §1.15); the outbound half violates the capability-honesty rule the target applies to every adapter (AC-35): simulated traffic is indistinguishable from live traffic in configuration, logs, and metrics.

**GAP-22 (rules outside domain layers).** §1.8 places rules in domain layers and limits controllers and UI components to translation and orchestration; today rules sit in oversized components and in activities' controllers. This is the only gap where a finding Doc 03 classified as technical debt (ARCH-034) counts — because Document 04 explicitly requires a different state for the rules-in-UI half; the *size* half remains debt (N-08).

Reservations appears in this register only as *evidence within* GAP-04, GAP-18, and GAP-12 — current-state facts about how other contexts touch it. Its internal design remains out of scope (Document 04 §3.4.6; N-07).

## 7. Open-Decision-Conditioned Gaps

Four entries cannot be stated as violations until a Document 04 open decision closes. None of them is an architectural failure today.

| ID | What is conditioned | OD | What the OD controls |
|---|---|---|---|
| GAP-23 | Whether the current trigger-coupled reporting path violates §6.5 depends on the chosen path: under read contracts the current ETL's coupling to authoritative tables is a violation; under rebuildable projections a projection pipeline is required and the current shape is simply not yet that. | OD-05 | Reporting/analytics data path |
| GAP-24 | Whether today's mixed `hotel_id` / `property_id` / `tenant_id` naming violates AC-02 depends on which concept is the isolation root and which single parameter expresses it. The *double-parameter* defect itself is already counted inside GAP-02. | OD-01 | Tenancy hierarchy and isolation root |
| GAP-25 | Whether the current Swagger arrangement satisfies AC-25 cannot be judged before the publication mechanism (OpenAPI, shared package, or both from one source) and its coverage of internal and integration surfaces are fixed. The doubled path it advertises is already counted in GAP-13. | OD-04 | Contract publication mechanism |
| GAP-26 | Which identifier format the target requires — and therefore whether the current mixed UUID/text/composite forms diverge from it — is OD-08's question. The *typing* violation (untyped cross-schema strings) is already counted in GAP-09. | OD-08 | Cross-domain identifier convention |

OD-02, OD-03, OD-07, OD-09, and OD-10 appear elsewhere only as **dependencies of confirmed gaps' expression** (GAP-02, GAP-11, GAP-09, GAP-03, GAP-15 respectively); OD-06 appears as the basis of a non-gap (N-02). No open decision is resolved here.

## 8. Non-Gaps & Explicitly Deferred Items

| Ref | Item | Why it is not a gap | Classification |
|---|---|---|---|
| N-01 | Sound architecture: single composition root and clean layer direction, DDD cores, CQRS bus with pipeline, disposable-schema migration-replay harness, verified OTA ingress, fail-closed auth ordering (ARCH-001…ARCH-006, ARCH-053) | The target explicitly preserves these (§1.15); current state already satisfies the corresponding rules | NOT A GAP |
| N-02 | Single-process, monolithic composition of contexts | The target runs as one system composed of contexts (§2.1); process topology is open (OD-06) and the target does not require separate processes | NOT A GAP |
| N-03 | One physical database and the *count* of existing schemas as such | The target's fixed minimum is per-context schema granularity (counted as GAP-09 where unmet); escalation to database-per-context is optional and open (OD-07) | NOT A GAP |
| N-04 | Admin and mobile prototypes with near-zero surface (ARCH-036) | Client shipping is a product/deployment decision; reach is scope-based (§2.5, §10.1.3). The target requires no minimum client surface | NOT A GAP |
| N-05 | Zero production imports of backend source or database clients from client code (Document 01 §8.5) | This is a target-compliant property today (§4.2.6) — recorded to preserve, not as a defect | NOT A GAP |
| N-06 | Test coverage targets, framework/runner selection, suite organisation | Not fixed by the target; explicitly open as OD-10. The structural verification rules are counted in GAP-15 | DEFERRED |
| N-07 | Reservations internal design, tables, endpoints, state set, readiness | Excluded by Document 03 §13 and Document 04 §3.4; this step starts no Reservations work | DEFERRED |
| N-08 | Component size (46 components over 500 lines) and an unconsumed shared pricing module (ARCH-034, size half) | No target rule constrains component size or module completeness; only the rules-in-UI half counts (GAP-22) | NOT A GAP |
| N-09 | Unused library packages with no contract surface (`@xylo/notifications` 0 imports, `@xylo/ui-native` null stubs, unimported `@xylo/config`) | These are packages, not published context surfaces; §1.9 dormancy attaches to subsystems/surfaces (those with surfaces are counted in GAP-12) | NOT A GAP |

## 9. Summary & Readiness for Correction Planning

### 9.1 Counts

| Classification | Count |
|---|---|
| `CONFIRMED` architectural gaps | **22** |
| `CONDITIONED BY OPEN DECISION` | **4** |
| `NOT A GAP` | **7** |
| `DEFERRED` | **2** |
| **Total register entries (GAP-01…GAP-26)** | **26** |

Severity of the 22 confirmed gaps: **4 CRITICAL** (GAP-01, GAP-02, GAP-08, GAP-20), **13 HIGH**, **5 MEDIUM**.

Consolidation: 53 current-state findings → 26 register entries; no finding was duplicated across confirmed gaps where a single architectural cause exists (one finding may serve as evidence in more than one row only where its aspects genuinely violate different rules, e.g. ARCH-026 ownership → GAP-08's DTO aspect → GAP-14).

### 9.2 Coverage of the priority areas

| Priority area | Register coverage |
|---|---|
| Tenant / property isolation | GAP-01, GAP-02 (+ GAP-24 conditioned) |
| Authorization boundaries | GAP-03 |
| Domain ownership | GAP-18 |
| Cross-domain dependencies | GAP-04, GAP-05 |
| Persistence / schema ownership | GAP-06, GAP-07, GAP-08, GAP-09, GAP-17 |
| API boundaries | GAP-13, GAP-14 (+ GAP-03 at the surface) |
| Frontend architectural boundaries | GAP-19, GAP-20 (+ GAP-22) |
| Integration boundaries | GAP-21 (inbound preserved — N-01) |
| Legacy paths violating target ownership | GAP-16 |
| Cross-cutting structure | GAP-10, GAP-11, GAP-12, GAP-15 |

Every priority area named for this step has at least one register entry; no area produced gaps beyond the evidence.

### 9.3 What this analysis establishes

- The confirmed gaps are **global in character**: isolation, dependency direction, persistence access, schema reproducibility, authorization uniformity, and verification affect all contexts — consistent with Document 03's four global preconditions (P1–P4), each of which is represented by at least one confirmed gap.
- Domain-specific gaps concentrate where Document 03's readiness verdicts are weakest (front-office, inventory, purchasing, billing, cashiering), but this document does **not** re-grade readiness — Document 03 §13 remains authoritative.
- Residual uncertainties R1–R9 (Document 03 §16) were checked against every classification: Document 03 states none of them changes any classification, and none changes any gap classification here.
- Where the target defers mechanism selection, this document records a dependency or a conditioned entry — **no open decision is resolved, and no gap is closed by assumption.**

### 9.4 What this document deliberately does not contain

No remediation plan, no correction sequence, no ordering of gaps for treatment, no work breakdown, no owners, no estimates, no implementation of any kind. Statements of the form "gap exists" are complete within this document; how and when any gap is addressed belongs to a subsequent correction-planning step, and remains unperformed.

---

**End of Document 05 — Architecture Gap Analysis.**

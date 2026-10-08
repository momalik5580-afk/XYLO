# Document 06 — Architecture Correction Plan

**Step 06 of the XYLO Project-Wide Architecture Review**

Status: COMPLETE — PLANNING ONLY (no implementation)
Follows: `docs/architecture/01_CURRENT_ARCHITECTURE_AUDIT.md`, `docs/architecture/02_OPEN_EVIDENCE_RESOLUTION.md`, `docs/architecture/03_ARCHITECTURE_FINDINGS_AND_VERDICT.md`, `docs/architecture/04_TARGET_ARCHITECTURE.md`, `docs/architecture/05_ARCHITECTURE_GAP_ANALYSIS.md`
Date: 2026-10-08

---

## 1. Purpose & Scope

This document is the **controlled correction plan** of the review: it converts the confirmed gap register of Document 05 into consolidated correction workstreams and correction items that move the repository from the audited current state (Documents 01–03) toward the locked target architecture (Document 04). Its single product is the Correction Register (§4), supported by its dependency model (§5), priority classes (§6), sequencing constraints (§7), verification strategy (§8), the Reservations gate (§9), and the deferred set (§10).

It does **not**: implement anything — no code, schema, migration, test, configuration, or CI change is made or instructed; estimate, schedule, assign, date, or break any correction into tasks or sprints; resolve any Document 04 Open Decision (OD-01…OD-10); assert any violation that Document 05 conditioned on an Open Decision (GAP-23…GAP-26); reopen Documents 01–05 or any classification, severity, readiness verdict, or residual within them; create new current-state findings — no new `ARCH-` identifier exists in this document and no claim outside Documents 01–05 is made; design, specify, or grade Reservations; or prescribe code-level changes — no file path, function name, endpoint path, line reference, or code edit appears anywhere in this document.

### 1.1 Inputs / Authority

| Input | Role in this plan |
|---|---|
| `01_CURRENT_ARCHITECTURE_AUDIT.md` | Current-state facts and evidence locations behind each correction's problem statement |
| `02_OPEN_EVIDENCE_RESOLUTION.md` | Resolved evidence items E-01…E-10 cited as current-state proof |
| `03_ARCHITECTURE_FINDINGS_AND_VERDICT.md` | Authoritative finding IDs (ARCH-001…ARCH-053), preconditions P1–P11, readiness verdicts, residuals R1–R9 |
| `04_TARGET_ARCHITECTURE.md` | Authoritative target rules: principles §1, boundaries §2–§6, isolation §7, schema §8, API §9, frontend §10, integrations §11, testing §13, legacy §14, constraints AC-01…AC-43 (§15), Open Decisions OD-01…OD-10 (Appendix A) |
| `05_ARCHITECTURE_GAP_ANALYSIS.md` | Authoritative gap register GAP-01…GAP-26 (22 confirmed, 4 conditioned) and non-gaps N-01…N-09 |

Documents 01–05 are unchanged by this step. Every correction in §4 cites its gap and its target constraints from these documents as *basis*; this document creates no finding and no new current-state claim.

### 1.2 Identifier conventions

| Identifier | Meaning |
|---|---|
| `WS-1`…`WS-8` | Correction workstreams (§3) — architectural concern families, not tasks |
| `C-01`…`C-24` | Correction items (§4) — planned architectural corrections, all with Status `PLANNED` |
| `SC-1`…`SC-11` | Sequencing constraints (§7) — dependency statements only, never a schedule |
| `BLOCKED BY OPEN DECISION — OD-xx` | Marker on a correction item whose completion requires a Document 04 Open Decision to close (§5.3) |

Priority classes P0–P4 (§6) are planning classes only. Status has exactly one value in this step: `PLANNED`.

**Section-reference convention.** A `§` citation attached to a *target constraint* — in the Target constraints column, in Problem and Intended outcome statements, and in principle or rule references such as §1.6, §7.4, §13.3 — is Document 04's numbering. A `§` citation to this document's own plan structure — the register (§4), dependencies (§5, §5.3), priorities (§6), sequencing (§7), the Reservations gate (§9), and the deferred set (§10.1, §10.2) — is this document's numbering; where the same number exists in both documents, the surrounding wording (deferred per, gate at, marker in, table at) identifies the intended document.

## 2. Planning Principles

Every correction in §4 is planned under these principles; a correction that cannot be stated without violating one is re-stated rather than planned.

- **Preserve working behavior.** Corrections change architectural state, not product behavior. Sound properties are protected, not reworked (Document 04 §1.15; N-01). Where a correction's execution would alter user-observable behavior, that change is called out as risk, never assumed away.
- **Smallest safe correction.** Each item addresses exactly what its register entries prove defective. Nothing is touched because it is nearby, old, or disliked.
- **Traceability before action.** Every item cites gap IDs, current-state finding IDs, target constraints, and — where applicable — a Document 03 precondition. No orphan work, no work that traces only to taste.
- **No implementation without verified prerequisites.** An item's preconditions (§5) must hold before it executes; where a prerequisite is an Open Decision, the item is marked, not started.
- **No ownership bypass.** Corrections never route around a context's owner; they restore ownership — single writer, owning persistence, owning contract.
- **No new cross-domain coupling.** A correction may remove a dependency; it may never introduce one outside contract surfaces (AC-06, AC-14).
- **Converge, do not accumulate.** One mechanism per concern (§1.6): corrections retire parallel mechanisms rather than add wrappers, adapters, or temporary second paths.
- **Capability honesty.** A subsystem is active or declared dormant, never silently inert (§1.9); a simulated behavior is labelled as simulated (AC-35); dormancy is declared before anything is deleted.
- **Incremental, never big-bang.** Corrections land in separable increments each of which leaves the repository coherent; no flag-day rewrite, no replace-everything step.
- **Machine enforcement over convention.** The target classifies convention as not-yet-an-architecture (§1.10); every item's outcome must end in a check that fails on regression, not in a documented intention.
- **Open-decision discipline.** No OD is resolved, pre-selected, or assumed here. Conditioned gaps stay unasserted; blocked items keep their unconditional invariant and stop at the decision boundary.
- **No speculative redesign.** No microservices split, no database-per-context mandate (OD-07 remains open), no event-driven rewrite, no process-topology change (OD-06 remains open), no framework or platform replacement as a correction.

## 3. Correction Workstreams

### 3.1 Workstream definitions

| WS | Workstream | Confirmed GAP coverage | Items | Target constraint family |
|---|---|---|---|---|
| WS-1 | Identity-derived scope and database isolation | GAP-01, GAP-02 | C-01, C-02 | Tenancy: AC-01, AC-03, AC-04, AC-05; §7 |
| WS-2 | Schema, migration and generated-client integrity | GAP-06, GAP-07, GAP-08, GAP-09 | C-03, C-04, C-05, C-06 | Schema: AC-18…AC-22, AC-43; §8 |
| WS-3 | Domain boundary, dependency direction and rule placement | GAP-04, GAP-05, GAP-10, GAP-17, GAP-22 | C-07, C-08, C-09, C-10, C-11 | Structure: AC-06, AC-07, AC-09, AC-13, AC-14, AC-15; §4, §6.3, §1.8 |
| WS-4 | Single authoritative ownership and fact paths | GAP-18, GAP-11 | C-12, C-13 | Data: AC-12, AC-16, AC-36, AC-38; §1.1, §3, §5 |
| WS-5 | API surface architecture | GAP-03, GAP-13, GAP-14 | C-14, C-15, C-16, C-17 | API: AC-23, AC-24, AC-26, AC-27; §9 |
| WS-6 | Frontend architecture | GAP-19, GAP-20 | C-18, C-19, C-20 | Client: AC-30, AC-31, AC-32, AC-33; §10 |
| WS-7 | Legacy containment and capability honesty | GAP-16, GAP-12, GAP-21 | C-21, C-22, C-23 | Legacy and integration: AC-35, AC-42, AC-43; §1.9, §11.3, §14 |
| WS-8 | Architectural guardrails and verification enforcement | GAP-15 | C-24 | Testing: AC-40, AC-41; §1.10, §13 |

### 3.2 Why consolidation, not one task per gap

The 22 confirmed gaps consolidate into **8 workstreams and 24 correction items** — not 22 items — for three reasons, each visible in the register:

1. **Shared cause, shared correction surface.** GAP-04 (internal imports) and GAP-05 (persistence reached outside owners) are the code-level and data-level halves of one boundary failure; they share one verification surface (boundary checks) and one dependency (contract surfaces must exist), so they sit in one workstream as separate items under one enforcement discipline. GAP-06, GAP-07, GAP-08 form a chain — baseline, change channel, client — where any one planned alone would be unverifiable alone.
2. **One gap, two constraints, different dependencies.** GAP-03 splits into C-14 (reachability of every mutating endpoint, AC-23, unconditional) and C-15 (one vocabulary per request, AC-24, OD-09-blocked): planning them as one item would mark a security-class correction blocked when its safety half is not. GAP-19 likewise splits into C-18 (server-state layer and contract-sourced types) and C-19 (machine-enforced feature boundaries), which have different preconditions and different verification.
3. **One item, several expressions.** GAP-18 consolidates five concept families with two or three writers each under one single-writer rule (AC-12); the item corrects the *rule*, and per-concept writer inventories are its verification — not eight sibling items.

Conditioned gaps (GAP-23…GAP-26) produce **no items**: they are deferred in §10.1 until their Open Decisions close.

### 3.3 Coverage map

| Workstream | Confirmed gaps | Items | Conditioned gaps noted (deferred per §10.1) |
|---|---|---|---|
| WS-1 | GAP-01, GAP-02 | 2 | GAP-24 (OD-01) |
| WS-2 | GAP-06, GAP-07, GAP-08, GAP-09 | 4 | — |
| WS-3 | GAP-04, GAP-05, GAP-10, GAP-17, GAP-22 | 5 | — |
| WS-4 | GAP-18, GAP-11 | 2 | — |
| WS-5 | GAP-03, GAP-13, GAP-14 | 4 | GAP-25 (OD-04) |
| WS-6 | GAP-19, GAP-20 | 3 | — |
| WS-7 | GAP-16, GAP-12, GAP-21 | 3 | — |
| WS-8 | GAP-15 | 1 | — |
| **Total** | **22 confirmed** | **24** | **4 conditioned** |

All 22 confirmed gaps are covered by at least one item; no item exists without a confirmed gap; no new gap is introduced.

## 4. Correction Register

The register is the primary artefact. Each item carries exactly these fields: Correction ID, Workstream, GAP IDs, ARCH IDs, Target constraints, Problem, Intended outcome, Preconditions / dependencies, Risk, Verification requirement, Status. Every item's Status is `PLANNED`. Problem statements restate Document 05 evidence; intended outcomes state architectural end-states, never edits.

### WS-1 — Identity-derived scope and database isolation

#### C-01 — Derive request scope from authenticated identity

| Field | Value |
|---|---|
| Workstream | WS-1 |
| GAP IDs | GAP-01 |
| ARCH IDs | ARCH-007 (Document 03 P1) |
| Target constraints | AC-01; AC-05; §7.1 |
| Problem | Scope for non-public requests is selected by a client-supplied header (`x-property-id`, per this step's evidence basis in ARCH-007) with property authorization disabled, instead of being derived from the authenticated principal; the client therefore influences the boundary that operates on every request path. |
| Intended outcome | Every non-public request's scope derives from authenticated identity; no client-supplied header, parameter, route value, or body field is authoritative for scope; an operation outside the principal's scope carries an explicit cross-property grant checked at the boundary (AC-05). |
| Preconditions / dependencies | Document 03 precondition P1. Unconditional: GAP-01's violation is independent of OD-01 (Document 05 §5). The single-root *expression* of scope (AC-02) is conditioned on OD-01 and tracked as GAP-24 (§10.1), not inside this item. |
| Risk | Over-broad identity-derived scope could deny legitimate cross-property operations (central reservations, corporate users); the explicit grant path must widen deliberately, never the default. The change sits on the request path of every endpoint, so preserved behavior must be proven per surface, not assumed. |
| Verification requirement | Executable isolation evidence that scope cannot be set by client input (including the header named above), that a principal's default scope holds, and that a cross-property operation without a grant is refused; evidence runs where its gate applies (§13.3). |
| Status | PLANNED |

#### C-02 — Declare and enforce database-level isolation for the connecting role

| Field | Value |
|---|---|
| Workstream | WS-1 |
| GAP IDs | GAP-02 (+ GAP-24 conditioned, deferred per §10.1) |
| ARCH IDs | ARCH-008, ARCH-051 (absorbed); Document 02 E-01; Document 03 P2 |
| Target constraints | AC-03; AC-04; AC-20; §7.4–7.5 |
| Problem | No effective database-level isolation exists in any environment this repository can produce: policies exist only live, under an owner-plus-superuser-bypass role, with zero forced enforcement, and setters write a scope parameter readers never read — so neither required enforcement layer (§7.4) functions, and application authorization and database isolation are not independent layers. |
| Intended outcome | Isolation policy is declared in this repository, applies to the exact role the application connects as, and is enforced by the storage engine — or its absence is an explicit, documented risk acceptance (AC-03); the two enforcement layers remain independent, so misconfiguring one cannot disable the other (AC-04); any database-resident behaviour involved is migration-owned (AC-20). |
| Preconditions / dependencies | **BLOCKED BY OPEN DECISION — OD-02** (database-level isolation mechanism selects the form compliance takes). Document 03 precondition P2. The invariant — declared in-repo, effective for the connecting role, layered independently — holds regardless of OD-02; only the mechanism that realises it waits. Depends on C-04 (declaration travels through the migration channel) and therefore on C-03. |
| Risk | Applying enforcement to the connecting role before the baseline is reproducible would compound drift or lock environments out; enforcement the application cannot satisfy would break live paths, so policy and application behaviour must be validated together in an environment produced by replay. |
| Verification requirement | Evidence that isolation declared in-repo is in force for the exact connecting role in an environment built by replaying migrations from zero; that each layer still functions when the other is misconfigured (AC-04); and that the scope parameter written by setters is read by readers. |
| Status | PLANNED |

### WS-2 — Schema, migration and generated-client integrity

#### C-03 — Recover a reproducible schema and migration baseline

| Field | Value |
|---|---|
| Workstream | WS-2 |
| GAP IDs | GAP-06 |
| ARCH IDs | ARCH-010, ARCH-044, ARCH-045, ARCH-050; Document 02 E-05; Document 03 P3 |
| Target constraints | AC-18; AC-22; AC-43; §8.1; §8.3 |
| Problem | Ledger, migration directories, and database disagree: 28 of 51 migrations untracked, two unfinished ledger rows, one table dropped outside history, one table referenced fifteen times and never created, an untracked schema source, and 14 live tables with no model. |
| Intended outcome | Migration ledger, migration directories, and the database under test agree — any disagreement fails the build (AC-18); every environment's schema is produced by replaying migrations from zero (AC-22); live tables without models are declared or fail the drift gate (AC-43). |
| Preconditions / dependencies | Document 03 precondition P3. No correction dependency — this item is a precondition for C-04, C-05, C-06 and for C-02's migration-declared policy. No open decision involved. |
| Risk | Reconciling three disagreeing records requires classifying each discrepancy as abandoned, unmodelled, or historically dropped; a wrong classification silently blesses drift instead of correcting it, and the classification itself changes what the baseline means. |
| Verification requirement | Executable proof that a fresh environment built by replay reaches the committed state; that ledger-to-directory-to-database comparison fails the build on disagreement; that every unmodelled live table is declared (drift gate). |
| Status | PLANNED |

#### C-04 — Make migrations the only channel for schema and database-resident behaviour

| Field | Value |
|---|---|
| Workstream | WS-2 |
| GAP IDs | GAP-07 |
| ARCH IDs | ARCH-020, ARCH-041, ARCH-052, ARCH-008; Document 02 E-01, E-02 |
| Target constraints | AC-19; AC-20; §8.5 |
| Problem | Two services execute table and index creation at boot; 386 policies and two data-mutating triggers exist with no source in any migration, so schema and database-resident behaviour change outside the change channel. |
| Intended outcome | All schema change occurs through migrations and no application code creates or alters schema at boot or at runtime (AC-19); all policies, triggers, and functions are declared in migrations and owned by the schema's owning context (AC-20). |
| Preconditions / dependencies | Depends on C-03 (the channel can only be trusted once replay agrees with itself). Interlocks with C-02 (isolation policy is declared through this channel) and C-21 (triggers on abandoned tables are declared here or retired). |
| Risk | Moving boot-time DDL and undeclared database objects into migrations makes previously implicit state explicit; drifted environments will fail replay until reconciled — the intended signal, but one that surfaces latent breakage in bulk. |
| Verification requirement | Evidence that no application startup path executes schema change; that every database-resident object in use is produced by migration replay; that a schema difference introduced outside migrations fails verification. |
| Status | PLANNED |

#### C-05 — Restore generated-client reproducibility

| Field | Value |
|---|---|
| Workstream | WS-2 |
| GAP IDs | GAP-08 |
| ARCH IDs | ARCH-046, ARCH-011; Document 02 E-06; Document 03 P5 |
| Target constraints | AC-21; §8.4 |
| Problem | The generated client in use is not reproducible from committed schema: a file-linked client declares column types against a different schema than live and migration state, carries four ghost models, and is imported at more than twenty production sites; schema-to-client skew exists silently. |
| Intended outcome | The generated database client is reproducible from committed schema; schema-to-client skew fails typecheck; exactly one client per schema exists (AC-21). |
| Preconditions / dependencies | Depends on C-03 (committed schema must be the true baseline) and C-04 (schema changes enter through one channel). Document 03 precondition P5 (inventory client reflects current schema). |
| Risk | Regenerating from the true schema exposes live column differences the current client hides; production imports of the skewed client must move together with regeneration or typecheck fails broadly — ordering must keep the build coherent at every intermediate point. |
| Verification requirement | Evidence that regeneration from committed schema is deterministic; that introduced skew fails typecheck; that no second client per schema is importable from application code. |
| Status | PLANNED |

#### C-06 — Meet per-context schema granularity and type cross-context keys

| Field | Value |
|---|---|
| Workstream | WS-2 |
| GAP IDs | GAP-09 (typing aspect also referenced by GAP-26, deferred per §10.1) |
| ARCH IDs | ARCH-009 |
| Target constraints | §8.8.1; §6.7.2 |
| Problem | Most contexts' models share one schema beside two named schemas, over one physical database — below the target's per-context granularity minimum — and cross-schema keys are untyped strings, so a key's referent is unchecked by the compiler. |
| Intended outcome | Every context meets the target's minimum schema granularity; no cross-schema reference remains an untyped string; any escalation beyond the minimum follows the granularity decision rather than assumption. |
| Preconditions / dependencies | **BLOCKED BY OPEN DECISION — OD-07** (persistence granularity: whether any context escalates beyond per-context schema) **and OD-08** (cross-domain identifier format for typed keys). The unconditional halves — minimum granularity per context, elimination of untyped cross-schema keys — proceed; escalation scope and final key format wait. Depends on C-03 and C-05 (baseline and reproducible client precede any split). |
| Risk | Schema splitting and key retyping are wide-reaching; without a reproduced baseline they compound drift, and guessing an identifier format would bake in a choice OD-08 must still make. |
| Verification requirement | Evidence that each context's models sit at or above the granularity minimum; that no cross-schema reference is an untyped string; that client regeneration after any split stays deterministic. |
| Status | PLANNED |

### WS-3 — Domain boundary, dependency direction and rule placement

#### C-07 — Route cross-context use through exported surfaces only

| Field | Value |
|---|---|
| Workstream | WS-3 |
| GAP IDs | GAP-04 |
| ARCH IDs | ARCH-013, ARCH-030, ARCH-029; Document 03 P4 |
| Target constraints | AC-06; AC-07; §4.3; §4.4; §5.1 |
| Problem | 78 cross-module internal-import edges exist — front-office reaches into reservations' internals across seven layers on 35 of them — plus a reverse infrastructure-into-modules edge and a mutual synchronous Availability-Rates cycle; contexts use each other's internals instead of exported surfaces. |
| Intended outcome | Cross-domain use occurs only through the surface each context exports, with no import reaching internal layers (AC-06); dependency direction follows the layer model with no reverse edges, no mutual synchronous dependence, and no registration-time cycles (AC-07). |
| Preconditions / dependencies | Document 03 precondition P4 (holds for every context; the reservations half is the §9 gate). The surfaces callers move onto must exist as single-sourced contracts (interlocks with C-17). No open decision involved. |
| Risk | Replacing internal imports with contract calls changes call timing and error surfaces; the mutual Availability-Rates cycle cannot be broken by moving imports alone — one direction must yield a contract, altering behavior for whichever side currently reaches across. |
| Verification requirement | Machine-checked boundary rules failing on internal-layer imports, reverse edges, and cycles; call-path evidence that former internal edges now traverse an exported surface. |
| Status | PLANNED |

#### C-08 — Confine persistence access to the owning context

| Field | Value |
|---|---|
| Workstream | WS-3 |
| GAP IDs | GAP-05 |
| ARCH IDs | ARCH-014; Document 01 (cross-schema audit write) |
| Target constraints | AC-14; §4.3.2–4.3.3; §5.1.3; §6.2 |
| Problem | 233 module files use the shared database client, 144 use raw SQL, six controllers inject it directly, the shared repository layer has two references, and a common audit module writes into another schema's tables — persistence is reached outside the owning context by many paths. |
| Intended outcome | No context reads another context's tables, repositories, or database client by any path other than contract invocation or an approved read model (AC-14); persistence access exists only inside the owning context's layer. |
| Preconditions / dependencies | Interlocks with C-12 (single writer defines who owns what), C-09 (transaction ownership stays with the writer), and C-24 (the boundary rule is machine-checked). Depends on contract or read-model paths existing where legitimate cross-context reads remain (AC-14's approved read model). |
| Risk | Cross-context reads often encode implicit joins and reporting shortcuts; removing them without an approved read model shifts load or breaks flows that silently depended on reading another owner's tables. |
| Verification requirement | Machine-checked rule that no module outside an owner reaches that owner's tables, repositories, or database client; documented approved read models for every cross-context read that legitimately remains. |
| Status | PLANNED |

#### C-09 — Establish one transaction per operation on money and inventory paths

| Field | Value |
|---|---|
| Workstream | WS-3 |
| GAP IDs | GAP-17 |
| ARCH IDs | ARCH-019; Document 03 P8 |
| Target constraints | AC-13; AC-15; §1.7; §6.3 |
| Problem | Check-in and check-out post money-affecting statements outside the command transaction, and four services perform multi-step writes with no transaction at all — operations commit partially. |
| Intended outcome | One application operation equals one transaction; multi-step writes affecting money or inventory complete within a single transaction boundary (AC-13); ledger and historical records stay append-only, with correction by new entry referencing the original (AC-15). |
| Preconditions / dependencies | Document 03 precondition P8 (money- and inventory-affecting paths). Depends on C-08 (the boundary belongs to the owning writer). No open decision involved. |
| Risk | Widening boundaries to cover previously separate writes increases lock hold time and converts partial success into full rollback; callers that tolerated partial completion will observe different failure behavior, which must be surfaced rather than absorbed silently. |
| Verification requirement | Executable evidence that multi-step money and inventory operations commit all-or-nothing; that no money-affecting statement executes outside its command's boundary; that history corrections arrive as new entries, never edits. |
| Status | PLANNED |

#### C-10 — Reduce cross-cutting concerns to one reachable mechanism each

| Field | Value |
|---|---|
| Workstream | WS-3 |
| GAP IDs | GAP-10 (communication and job family separated into C-13) |
| ARCH IDs | ARCH-015, ARCH-017, ARCH-021, ARCH-037 |
| Target constraints | AC-09; AC-28; §1.6 |
| Problem | Two command buses with identical tokens, two audit systems, two policy engines, three idempotency paths, two error types, four HTTP client stacks, and two configuration roots with two feature-flag systems — the one-mechanism rule is violated across cross-cutting concerns, including configuration. |
| Intended outcome | Exactly one mechanism exists and is reachable per cross-cutting concern — command bus, audit, policy engine, idempotency, error type, HTTP client, configuration, cache, scope context (AC-09); retryable commands carry idempotency at the boundary under that single mechanism (AC-28). |
| Preconditions / dependencies | Direction of convergence is determinable from the repository's own reachability evidence — no Appendix A decision governs these concerns. The communication and job family is out of this item's execution scope and is planned as C-13, which is blocked by open decision OD-03 (§5.3). Where a concern encodes ownership, C-12's designation precedes retirement of the parallel mechanism. |
| Risk | Retiring a parallel mechanism removes call sites that currently work; identical tokens across two buses make the wrong survivor easy to choose, and a wrong survivor silently drops registrations rather than failing loudly. |
| Verification requirement | Machine-checked single-mechanism rule per concern; reachability evidence that the surviving mechanism carries every current registration; proof that retired tokens and second configuration roots are no longer importable. |
| Status | PLANNED |

#### C-11 — Return business rules to domain layers

| Field | Value |
|---|---|
| Workstream | WS-3 |
| GAP IDs | GAP-22 |
| ARCH IDs | ARCH-034 (rules-in-UI half); ARCH-014 with Document 03 §13.2 (activities controllers) |
| Target constraints | §1.8; §4.1 (presentation row) |
| Problem | Business rules live inside 46 oversized UI components, and activities' behaviour lives in controllers over raw SQL — rules are held outside domain layers on both the client and server side. |
| Intended outcome | Rules sit in domain layers; controllers and UI components are limited to translation and orchestration (§1.8); activities' behaviour is expressed through its domain rather than through controllers. |
| Preconditions / dependencies | The activities half is constrained by C-08 (persistence reachable only from the owning layer); the UI half sits behind the client structure established by C-18 and C-19. No open decision involved. |
| Risk | Moving rules out of components changes where validation appears to users; where a rule is currently trusted in two places, the move must not leave two diverging copies mid-transition. |
| Verification requirement | Evidence that domain layers own the behaviour and that controllers and components contain orchestration only; behavioural parity for user-visible validation outcomes before and after the move. |
| Status | PLANNED |

### WS-4 — Single authoritative ownership and fact paths

#### C-12 — Establish one authoritative store per concept

| Field | Value |
|---|---|
| Workstream | WS-4 |
| GAP IDs | GAP-18 |
| ARCH IDs | ARCH-011, ARCH-012, ARCH-025, ARCH-026; Document 03 P6, P9, P10 |
| Target constraints | AC-12; AC-16; §1.1; §3.1; §3.3 |
| Problem | Several concepts have two or three live authoritative stores: dual inventory worlds, dual reservation and triple guest sources of truth, two owners for property and company, folio data written from outside billing, and six or more capacity counter locations. |
| Intended outcome | Every business concept has exactly one authoritative store and one owning context, with no second writer (AC-12); derived read models are explicitly non-authoritative and single-producer (AC-16). |
| Preconditions / dependencies | Gates Document 03 P6 (purchasing, stock, warehouse), P9 (guest identity), P10 (folio and charge data). Which store survives per concept is domain decision work — not an Appendix A open decision — but the designation must be recorded before C-21 executes and before job ownership is assigned inside C-13. |
| Risk | Choosing the survivor per concept is a data-reconciliation trigger; writers converge only after state reconciles, and mid-convergence dual writes must not be mistaken for compliance with the single-writer rule. |
| Verification requirement | Per-concept writer inventory showing exactly one authoritative store; evidence that second writers no longer accept writes; read models labelled non-authoritative wherever they exist. |
| Status | PLANNED |

#### C-13 — Reduce communication and job execution to one reachable path

| Field | Value |
|---|---|
| Workstream | WS-4 |
| GAP IDs | GAP-11 |
| ARCH IDs | ARCH-018, ARCH-032; Document 02 E-09 |
| Target constraints | AC-36; AC-38; §5.3.6; §5.6; §11.5 |
| Problem | An in-process dispatcher with zero registrations, a second outbox processor with no callers, an unregistered consumer, three overlapping workflow layers whose single workflow was never registered, and queues nothing listens on — only the events queue carries traffic. Parallel carriers exist and none demonstrably carries facts. |
| Intended outcome | Exactly one job and queue execution mechanism exists; every job belongs to exactly one owning domain; defined-but-unregistered and registered-but-unreachable jobs do not exist (AC-36); fact publication occurs inside the owning context's transaction boundary with exactly one publication path that demonstrably carries facts (AC-38). |
| Preconditions / dependencies | **BLOCKED BY OPEN DECISION — OD-03** (cross-domain communication and job-execution mechanism selection decides which stack survives). The invariants — no parallel carriers, no unreachable jobs, publication inside the owning transaction — hold now; survivor selection waits for the decision. Interlocks with C-10 (same one-mechanism family) and C-12 (publication path belongs to the owner). |
| Risk | Converging before OD-03 closes would commit the repository to a mechanism the target may not select; conversely, waiting while five carriers coexist keeps AC-36 and AC-38 unmet — so the unconditional portion (retiring carriers that carry nothing, registering what must run) must proceed without pre-selecting a winner. |
| Verification requirement | Evidence that exactly one mechanism carries traffic; that every registered job is reachable and every required job registered; that fact publication occurs within the owning context's transaction. |
| Status | PLANNED |

### WS-5 — API surface architecture

#### C-14 — Make authorization reachability an invariant of every mutating endpoint

| Field | Value |
|---|---|
| Workstream | WS-5 |
| GAP IDs | GAP-03 (reachability aspect) |
| ARCH IDs | ARCH-016; Document 02 E-03; Document 03 P7 |
| Target constraints | AC-23; §9.6 |
| Problem | Both permission vocabularies go inert when their decorator is absent, so a mutating endpoint can be reachable with no effective authorization requirement at all; nothing in the repository prevents that state. |
| Intended outcome | Every mutating endpoint carries exactly one authorization requirement with unambiguous semantics, and an endpoint without one is not reachable (AC-23). |
| Preconditions / dependencies | Document 03 precondition P7 — gates any new API surface. Reachability enforcement is unconditional; the vocabulary that evaluates the requirement belongs to C-15, which is blocked by open decision OD-09 (§5.3). Interlocks with C-16 (prefix) and C-17 (typed bodies) for contract-level checking. |
| Risk | Fail-closed enforcement will expose endpoints that currently rely on absent decorators; each must be corrected to carry a requirement rather than exempted, or exemption becomes the new silent state. |
| Verification requirement | Machine-checked rule that every mutating endpoint declares exactly one authorization requirement and that missing declarations fail the build; reachability probe demonstrating that an undeclared endpoint is not servable. |
| Status | PLANNED |

#### C-15 — Evaluate exactly one permission vocabulary and mechanism per request

| Field | Value |
|---|---|
| Workstream | WS-5 |
| GAP IDs | GAP-03 (uniformity aspect) |
| ARCH IDs | ARCH-016; Document 02 E-03; Document 03 P7 |
| Target constraints | AC-24; §1.5; §9.6 |
| Problem | Two disjoint permission vocabularies (47 and 173 sites) both evaluate on every request, producing ambiguous, overlapping semantics — no single vocabulary is authoritative. |
| Intended outcome | Exactly one permission vocabulary and one authorization mechanism are evaluated per request (AC-24). |
| Preconditions / dependencies | **BLOCKED BY OPEN DECISION — OD-09** (authorization mechanism and permission vocabulary selection decides which vocabulary survives). Depends on C-14 landing first, so that whatever vocabulary is selected, reachability already fails closed during convergence. Document 03 precondition P7 for the vocabulary half. |
| Risk | Merging vocabularies changes effective permissions for every principal; permissions that look equivalent across the two sets may not be semantically equal, and a naive union would silently widen access rather than unify it. |
| Verification requirement | Evidence that one vocabulary is evaluated per request across all surfaces; that retired vocabulary tokens are unreachable; that permission outcomes for representative principal classes are unchanged after convergence. |
| Status | PLANNED |

#### C-16 — Publish public routes at a single version prefix

| Field | Value |
|---|---|
| Workstream | WS-5 |
| GAP IDs | GAP-13 |
| ARCH IDs | ARCH-048; Document 02 E-04 |
| Target constraints | AC-27; §9.1.3 |
| Problem | Public routes resolve at a doubled version prefix on 21 controllers, the advertised contract shows the doubled path, and no consumer uses either form — the route surface is ambiguous. |
| Intended outcome | Public routes are versioned and prefixed exactly once; no doubled or ambiguous prefix exists (AC-27); the published contract advertises the form actually served. |
| Preconditions / dependencies | Unconditional. Interlocks with C-17 (contract advertisement matches served routes); contract *publication mechanism* remains conditioned (GAP-25, OD-04, §10.1) but the single-prefix invariant does not depend on it. |
| Risk | A consumer outside the repository's audit could depend on the doubled form; contract and route surface must move together, or the contract advertises a path that is not served. |
| Verification requirement | Evidence that each public route resolves at exactly one prefix; that contract output matches served paths; that route-resolution checks fail on a doubled prefix. |
| Status | PLANNED |

#### C-17 — Define contract types once and type every request body

| Field | Value |
|---|---|
| Workstream | WS-5 |
| GAP IDs | GAP-14 (+ GAP-25 conditioned, deferred per §10.1) |
| ARCH IDs | ARCH-033, ARCH-026 (DTO aspect) |
| Target constraints | AC-26; §9.4 |
| Problem | Two parallel reservation DTO sets are both consumed, duplicate property and company DTOs exist, and 30 of 87 controllers accept untyped request bodies — contract types are duplicated at the boundary and bodies are unchecked. |
| Intended outcome | Contract types are defined once at the owning boundary; duplicated DTO sets and untyped request bodies do not exist (AC-26); client-side domain types are sourced from these contracts (with C-18, AC-31). |
| Preconditions / dependencies | Feeds C-07 (single-sourced surfaces are what cross-context callers move onto) and C-14 (typed endpoints carry declared authorization). Publication format remains conditioned (GAP-25, OD-04) — this item unifies *type sources*, not the publication mechanism. No open decision blocks it. |
| Risk | Unifying parallel DTO sets surfaces where the two sets diverged — including fields one set validates and the other does not; typing previously untyped bodies rejects requests that were silently accepted, possibly from current consumers. |
| Verification requirement | Machine-checked single-definition rule for contract types at owning boundaries; type gate proving no controller accepts an untyped request body; a divergence report between formerly parallel DTO sets before unification. |
| Status | PLANNED |

### WS-6 — Frontend architecture

#### C-18 — Give each client exactly one server-state layer with contract-sourced types

| Field | Value |
|---|---|
| Workstream | WS-6 |
| GAP IDs | GAP-19 (server-state and types aspect) |
| ARCH IDs | ARCH-021, ARCH-023, ARCH-035 (host resolution aspect) |
| Target constraints | AC-30; AC-31; §10.2–10.3 |
| Problem | Four HTTP clients, five incompatible query-key registries, 13 of 20 stores fetching server state, double-cached endpoints with no shared invalidation, three independent domain types for one reservation concept, and nine hardcoded host fallbacks — there is no single server-state layer per application. |
| Intended outcome | Each application has exactly one server-state layer, separate from UI state, with one cache and invalidation path per endpoint (AC-30); domain types come from the published contract rather than client-local shapes (AC-31); endpoints resolve through one configuration source rather than hardcoded fallbacks. |
| Preconditions / dependencies | Depends on C-17 (contract-sourced types must exist before they can be sourced). Interlocks with C-19 (boundaries are enforced over this layer) and C-20 (auth state is placed within the single handling path). No open decision involved. |
| Risk | Migrating stores off four client stacks onto one changes cache semantics; endpoints double-cached today may show stale or missing data until invalidation paths converge, and host-fallback removal changes which endpoint resolves in misconfigured environments. |
| Verification requirement | Machine-checked single server-state layer per application; proof that no client-local domain shape duplicates a contract type; invalidation evidence showing exactly one path per endpoint and one host source per environment. |
| Status | PLANNED |

#### C-19 — Machine-enforce frontend feature boundaries

| Field | Value |
|---|---|
| Workstream | WS-6 |
| GAP IDs | GAP-19 (feature-boundary aspect) |
| ARCH IDs | ARCH-022, ARCH-035 (boundary reach aspect); Document 03 (lint coverage) |
| Target constraints | AC-32; §10.4 |
| Problem | A legacy-to-feature import cycle exists with 19 application files reaching into feature code, no feature boundary is enforced anywhere, and lint covers 209 of 871 web files — feature boundaries are conventional only, and convention is not architecture (§1.10). |
| Intended outcome | Feature boundaries are machine-enforced, and lint covers all application code of every client (AC-32); the legacy-feature import cycle no longer exists. |
| Preconditions / dependencies | Depends on C-18 (boundaries are checkable once the layer structure exists) and on C-24 (the checks run inside the type and lint gates). No open decision involved. |
| Risk | Enforcing boundaries over previously free imports fails first on the existing cycle; expanding lint coverage surfaces long-hidden violations in bulk, which must be corrected rather than suppressed. |
| Verification requirement | Machine-checked feature-boundary rules failing on cross-feature and legacy-into-feature imports; lint applicability proven over all application code of every client, with violations failing rather than warning. |
| Status | PLANNED |

#### C-20 — Remove client-side secrets and centralize client auth handling

| Field | Value |
|---|---|
| Workstream | WS-6 |
| GAP IDs | GAP-20 |
| ARCH IDs | ARCH-047, ARCH-035 (auth and cookie aspect) |
| Target constraints | AC-33; §10.6 |
| Problem | The JWT signing secret is inlined into the client bundle with hardcoded fallbacks, auth logic is duplicated across web and admin, and cookie access exists in three copies — token integrity is void because any holder of the bundle can forge what the server trusts. |
| Intended outcome | No credential, signing secret, or long-lived key exists in any client bundle, including fallback values; each application handles auth state in exactly one place (AC-33). |
| Preconditions / dependencies | Unconditional security-class work; no open decision and no correction prerequisite — it must not wait behind structural frontend work. Interlocks with C-18 (auth state placement) only for the consolidation half. |
| Risk | Removing an inlined secret invalidates any flow that depended on client-side signing; rotating rather than merely removing changes every issued token, so server and client behavior must move in lockstep or sessions break en masse. |
| Verification requirement | Bundle-level evidence that no signing secret, credential, or long-lived key — including fallback values — is present in any client build; machine check showing exactly one auth-state handling path per application. |
| Status | PLANNED |

### WS-7 — Legacy containment and capability honesty

#### C-21 — Contain legacy stores and pipelines

| Field | Value |
|---|---|
| Workstream | WS-7 |
| GAP IDs | GAP-16 |
| ARCH IDs | ARCH-043, ARCH-041, ARCH-052; Document 02 E-02; Document 03 P6 |
| Target constraints | AC-42; AC-43; §1.13; §6.6; §14.2–14.4 |
| Problem | `public.inventory_*` is still written by purchasing, procurement, housekeeping, and stock/receiving alongside the current inventory world with no bridge between them, and data-mutating triggers sit on abandoned tables, absent from every repository file — two pipelines run in parallel, undeclared. |
| Intended outcome | Legacy stores, pipelines, and paths receive no new behaviour, endpoints, or consumers; where two pipelines exist exactly one is authoritative (AC-42); legacy status is declared where the repository records its inventory, and undocumented unmodelled tables fail the drift gate (AC-43). |
| Preconditions / dependencies | Depends on C-12 (the authoritative store per concept must be designated first) and C-04 (surviving or retiring triggers happens through the migration channel). Declaration-before-containment follows C-22. Document 03 precondition P6. No Appendix A open decision involved. |
| Risk | Stopping the second writer while consumers still read from it breaks those consumers; triggers on abandoned tables may be the only mechanism maintaining derived state, and removing them silently stops that maintenance. |
| Verification requirement | Writer inventory showing exactly one pipeline accepting writes per concept; declared legacy inventory covering every legacy store and trigger; evidence that no new endpoint or consumer targets a legacy path. |
| Status | PLANNED |

#### C-22 — Declare every inert subsystem active or dormant

| Field | Value |
|---|---|
| Workstream | WS-7 |
| GAP IDs | GAP-12 |
| ARCH IDs | ARCH-027, ARCH-028, ARCH-042; Document 02 E-07 |
| Target constraints | §1.9; §2.7.3; §13.6; §14.6 |
| Problem | 19 of 37 modules are stubs or thin shells with two unregistered, a platform surface carries 173 permission annotations and zero first-party consumers, and 82 files have no non-test importers — inert subsystems are neither active nor declared dormant. |
| Intended outcome | Every subsystem with a surface is either active or explicitly declared dormant in the repository's inventory, with nothing silently inert (§1.9, §2.7.3); unregistered and unmodelled structure is reconciled or declared. |
| Preconditions / dependencies | Declaration precedes deletion and informs C-21 (containment) and C-15 (a surface with zero consumers affects vocabulary convergence choices). Dormancy declarations are statements of state, not decisions to delete — deletion is out of scope everywhere in this plan. No open decision involved. |
| Risk | Declaring a subsystem dormant invites later deletion on a wrong assumption; a subsystem with hidden consumers — test-only or external — would be misclassified, so declaration must rest on importer evidence rather than absence of feature work. |
| Verification requirement | Inventory showing every surface classified active or dormant with importer evidence; registration checks for modules expected active; zero unclassified stubs and zero unregistered modules. |
| Status | PLANNED |

#### C-23 — Distinguish simulated outbound behaviour from live in every signal

| Field | Value |
|---|---|
| Workstream | WS-7 |
| GAP IDs | GAP-21 |
| ARCH IDs | ARCH-038 |
| Target constraints | AC-35; §1.9; §11.3 |
| Problem | The payment gateway stub is hardwired, push SDKs are absent, and channel metrics are fabricated — operators cannot separate live from simulated traffic in configuration, logs, or metrics. |
| Intended outcome | Simulated outbound behaviour is distinguishable from real behaviour in every telemetry signal it emits (AC-35); each adapter's state — live, simulated, or dormant — is explicit (§1.9). |
| Preconditions / dependencies | Unconditional. Interlocks with C-22 (labelling is the integration-domain expression of capability honesty). Document 03 channels constraint (ARCH-038). No open decision involved. |
| Risk | Labelling fabricated metrics changes dashboards and alerts that currently treat them as real; downstream consumers of those signals will see the correction, which is the intended effect but must be visible through the signal itself rather than communicated out-of-band. |
| Verification requirement | Telemetry evidence that simulated adapters emit identifying labels in logs, metrics, and traces; configuration proof that simulated and live paths are explicitly selectable rather than hardwired. |
| Status | PLANNED |

### WS-8 — Architectural guardrails and verification enforcement

#### C-24 — Enforce the target's verification contract by machine

| Field | Value |
|---|---|
| Workstream | WS-8 |
| GAP IDs | GAP-15 |
| ARCH IDs | ARCH-024, ARCH-040, ARCH-039, ARCH-049, ARCH-031; Document 02 E-10; Document 03 P11 |
| Target constraints | AC-40; AC-41; §1.10; §13.3–13.5 |
| Problem | Database-gated suites skip on every pull request because the gate's prerequisites are absent from CI, 26 of 37 modules are untested, tests are excluded from the type gate, no boundary or architectural check exists anywhere, repository lint produces no findings, and the documented end-to-end command points at files that do not exist — the target's verification contract is unmet, so every other correction in this register would rest on convention. |
| Intended outcome | Database-gated suites run where their gate applies and silent skipping does not constitute a pass; test sources are included in the type gate (AC-40); a behaviour change carries executable evidence that fails on regression (AC-41); the boundary and architectural rules of this plan are checked by machine (§1.10). |
| Preconditions / dependencies | Boundary-specific checks (for C-07, C-08, C-19) land after the boundaries they check reach their corrected shape; the type-gate and CI-suite components have no correction dependency and are P1. Strategy specifics — coverage targets, framework and runner selection, suite organisation — are **deferred to OD-10** (§10.2), not blocked: the structural rules above hold regardless (GAP-15's classification). Interlocks with every other item — without this one, all register outcomes are convention (Document 05 §5 interlock note). |
| Risk | Turning silent skips into failures initially fails pipelines that currently pass; admitting test sources into the type gate surfaces type errors across existing tests in bulk; architectural checks introduced before their boundary stabilises would churn. |
| Verification requirement | CI evidence that gated suites execute where the gate applies and that a missing gate fails rather than passes; type gate demonstrably including test sources; at least one architectural check per boundary rule in this plan, shown failing on a violation. |
| Status | PLANNED |

## 5. Dependency Model

### 5.1 Correction-to-correction dependencies

| From | To | Why the order holds |
|---|---|---|
| C-03 | C-04, C-05, C-06 | A change channel, a regenerated client, and any granularity change are only verifiable against a baseline that replays to itself |
| C-04 | C-02 | Isolation policy must be declared through the migration channel (AC-20); with C-03 preceding C-04, the chain is C-03 → C-04 → C-02 |
| C-04 | C-21 | Triggers on abandoned tables are declared through the migration channel or retired through it |
| C-12 | C-21 | The authoritative store per concept must be designated before any second writer is contained |
| C-12 | C-13 | Jobs and facts belong to owning domains (AC-36, AC-38); ownership precedes path convergence |
| C-14 | C-15 | Reachability fails closed before vocabulary convergence, so no endpoint is unprotected mid-transition |
| C-17 | C-07, C-18 | Single-sourced contract types are the surfaces cross-context callers move onto and the types clients source |
| C-08 | C-09 | The transaction boundary belongs to the owning writer; confinement precedes boundary correction |
| C-07, C-08, C-19 | C-24 (boundary checks) | Architectural checks land after the boundary shape they check is settled |
| C-10 | C-13 | The communication and job family is the OD-03-governed member of the same one-mechanism concern |
| C-22 | C-21, C-15 | Declaration of active versus dormant precedes containment and informs vocabulary convergence |

### 5.2 Precondition gates (Document 03 P1–P11)

| Precondition (Document 03) | Corrections that discharge it | Target constraints |
|---|---|---|
| P1 — scope from authenticated identity | C-01 | AC-01, AC-04, AC-05 |
| P2 — isolation policy declared in-repo for the connecting role | C-02 | AC-03, AC-04, AC-20, AC-39 (mapping per Document 04 §15.9) |
| P3 — schema and migration baseline recoverable and consistent | C-03, C-04, C-05 | AC-18, AC-19, AC-21, AC-22, AC-43 |
| P4 — cross-domain use through exported surface | C-07 (with C-08 for data) | AC-06, AC-07 |
| P5 — inventory client reflects current schema | C-05 | AC-21 |
| P6 — single authoritative store per purchasing, stock, warehouse concept | C-12, C-21 | AC-12, AC-42 |
| P7 — every mutating endpoint carries an unambiguous authorization requirement | C-14, C-15 | AC-23, AC-24 |
| P8 — multi-step money or inventory writes in one transaction | C-09 | AC-13, AC-15 |
| P9 — single authoritative guest identity record | C-12 | AC-12 |
| P10 — folio and charge data has a single writer | C-12 (with C-09) | AC-12, AC-13 |
| P11 — behaviour change carries executable evidence | C-24 | AC-40, AC-41 |

### 5.3 Open-decision dependencies

Four items carry the marker **BLOCKED BY OPEN DECISION**. For each, the unconditional invariant proceeds and the mechanism or survivor selection waits; no decision is pre-selected here.

| Correction | OD | What the OD controls | What proceeds without it |
|---|---|---|---|
| C-02 | OD-02 | The mechanism by which database-level isolation is declared and enforced | Declaration in-repo for the connecting role, independent enforcement layers (AC-04), no runtime DDL |
| C-06 | OD-07 (and OD-08) | Whether any context escalates beyond per-context schema; the identifier format for typed keys | Minimum per-context granularity; elimination of untyped cross-schema keys |
| C-13 | OD-03 | Which communication and job-execution mechanism survives | Retirement of carriers that carry nothing, registration and reachability of required jobs, publication inside the owning transaction |
| C-15 | OD-09 | Which authorization mechanism and permission vocabulary are authoritative | One-vocabulary-per-request invariant; fail-closed reachability via C-14 |

Items that are **not** blocked despite open-decision adjacency, and why: **C-01** — GAP-01's violation is independent of OD-01 (Document 05 §5); the naming question is conditioned GAP-24, deferred (§10.1). **C-24** — strategy specifics await OD-10, but every structural rule it enforces holds regardless. **C-16, C-17** — contract *publication mechanism* is conditioned GAP-25 (OD-04); the single-prefix and single-type-source invariants do not depend on it. Conditioned gaps GAP-23, GAP-24, GAP-25, GAP-26 produce no items at all (§10.1).

## 6. Priority Classes

### 6.1 Class definitions

| Class | Definition | Assignment rule |
|---|---|---|
| **P0** | Security or architectural-safety blocker: the defect lets a client or an accident choose security-relevant state, or removes a safety layer entirely | Assigned only where continued operation is itself the hazard |
| **P1** | Global precondition or foundation: without it, other corrections cannot be verified or safely executed | Tied to Document 03 P1–P4, P11 and to the verification chain |
| **P2** | Structural correctness violating a target rule that holds unconditionally, but which does not block verification of P0/P1 work | Unconditional rules with execution risk or partial OD dependency |
| **P3** | Containment and declaration work whose timing depends on ownership designations and which carries the largest behavioral-change surface | Requires C-12's per-concept designations first |
| **P4** | Discretionary: correct only where convenient | Defined for completeness; **no item qualifies** — every item traces to a confirmed gap whose target rule holds unconditionally, so none is optional |

### 6.2 Distribution

| Class | Count | Items |
|---|---|---|
| P0 | **4** | C-01, C-02, C-14, C-20 |
| P1 | **10** | C-03, C-04, C-05, C-07, C-08, C-09, C-10, C-12, C-15, C-24 |
| P2 | **8** | C-06, C-11, C-13, C-16, C-17, C-18, C-19, C-23 |
| P3 | **2** | C-21, C-22 |
| P4 | **0** | none (see 6.1) |
| **Total** | **24** | C-01…C-24 |

P0 assignments and their evidence: **C-01** client-selected request scope (ARCH-007, GAP-01, CRITICAL); **C-02** absent database defence layer (ARCH-008, GAP-02, CRITICAL); **C-14** endpoints reachable with no effective authorization (ARCH-016, GAP-03's reachability aspect); **C-20** signing secret in the client bundle (ARCH-047, GAP-20, CRITICAL). Priority is a planning class only: **priority is not implementation order** — sequencing is governed by §7.

### 6.3 Blocked items

| Correction | Class | Blocker | Blocked portion |
|---|---|---|---|
| C-02 | P0 | OD-02 | Isolation mechanism selection only; the P0 invariant proceeds |
| C-06 | P2 | OD-07, OD-08 | Escalation scope and key format only; minimum granularity and key typing proceed |
| C-13 | P2 | OD-03 | Survivor mechanism selection only; carrier retirement and reachability proceed |
| C-15 | P1 | OD-09 | Vocabulary selection only; the one-vocabulary-per-request invariant proceeds |

**4 of 24 items** carry the marker BLOCKED BY OPEN DECISION.

### 6.4 Document tally

| Tally | Count |
|---|---|
| Correction workstreams (WS-1…WS-8) | 8 |
| Correction items (C-01…C-24) | 24 |
| P0 / P1 / P2 / P3 / P4 | 4 / 10 / 8 / 2 / 0 |
| Items marked BLOCKED BY OPEN DECISION | 4 |
| Confirmed gaps covered | 22 of 22 |
| Conditioned gaps deferred (§10.1) | 4 |

## 7. Sequencing Constraints

Statements of dependency only. No constraint here is a schedule: none carries a date, duration, owner, or task boundary.

1. **SC-1** Scope derivation (C-01) precedes any expansion of request-scoped API surface — Document 03 P1 applies to every such surface.
2. **SC-2** The reproducible baseline (C-03) precedes schema-channel work (C-04), client regeneration (C-05), and granularity work (C-06); each depends on replay agreeing with committed state.
3. **SC-3** Isolation declaration (C-02) enters only through the migration channel (C-04), so its declared form lands after C-04; its mechanism-dependent portion does not start before OD-02 closes.
4. **SC-4** Cross-context access moves onto exported surfaces (C-07, C-08) before any new cross-context call or data read is added — including every future Reservations surface (§9).
5. **SC-5** The boundary and architectural checks inside C-24 are introduced after the boundaries they check (C-07, C-08, C-19) reach their corrected shape; C-24's type-gate and CI-suite components depend on no other correction and may establish first.
6. **SC-6** Authorization reachability (C-14) precedes vocabulary convergence (C-15) so no endpoint is left unprotected mid-transition; no new mutating endpoint is added before C-14 holds — Document 03 P7.
7. **SC-7** Transaction-boundary work (C-09) precedes any extension of a money- or inventory-affecting multi-step write — Document 03 P8.
8. **SC-8** Authoritative-store designation (C-12) precedes legacy containment (C-21) and job-ownership assignment within C-13; second writers stop only after the survivor is authoritative — Document 03 P6, P9, P10.
9. **SC-9** Contract-type unification (C-17) precedes client layer consolidation (C-18); layer consolidation precedes feature-boundary enforcement (C-19).
10. **SC-10** OD-blocked items (C-02, C-06, C-13, C-15) execute only their unconditional invariants until their open decisions close; no mechanism or survivor is pre-selected.
11. **SC-11** No correction begins by replacing a working path wholesale: each preserves the properties Document 04 §1.15 and N-01 protect and changes only what its register entries prove defective.

## 8. Verification Strategy

### 8.1 Rules

Verification is part of the architecture, not a phase after it (§1.10). For this plan: silent skipping is not a pass and test sources belong in the type gate (AC-40); a behaviour change carries executable evidence that fails on regression (AC-41); evidence runs where its gate applies; and each item's Verification requirement (§4) names an evidence *class* from §8.2 — never a test, file, or command. No test, harness, or CI job is written in this step. Strategy specifics — frameworks, coverage targets, suite organisation — remain OD-10's (§10.2).

### 8.2 Verification categories

| Area | Verification category | Corrections covered |
|---|---|---|
| Tenant isolation | Identity-derivation and negative spoofing evidence; cross-property grant positive and negative evidence; isolation enforcement proof inside a replay-built disposable environment | C-01, C-02 |
| Schema integrity | Ledger-directory-database agreement gate; replay-from-zero build; client regeneration determinism; drift gate on unmodelled objects | C-03, C-04, C-05, C-06 |
| Boundaries | Machine-checked import rules: internal-layer imports, reverse edges, cycles, persistence confinement, feature boundaries, lint coverage over all client code | C-07, C-08, C-19 |
| Domain correctness | Transaction atomicity evidence; append-only ledger checks; per-concept single-writer inventories | C-09, C-12, C-21 |
| One-mechanism | Reachability and registration checks per concern; retired-token import failures; single configuration root | C-10, C-13 |
| API surface | Authorization-presence gate on every mutating endpoint; single-prefix route resolution; single-source contract types; typed-body gate | C-14, C-15, C-16, C-17 |
| Frontend | Single server-state layer check; contract-type sourcing check; single invalidation path per endpoint; bundle secret scanning | C-18, C-20 |
| Capability honesty | Inventory classification with importer evidence; telemetry label assertions distinguishing simulated from live | C-22, C-23 |
| Guardrail wiring | CI execution of gated suites with failure on missing gate; type gate including test sources; one architectural check per boundary rule, demonstrated failing on violation | C-24 (and all items) |

### 8.3 Evidence rules

Every item's outcome is closed only on evidence from its category, never on convention, review, or assertion. Where a category cannot yet run because its gate does not exist, establishing that gate is itself part of C-24 and precedes closure of the items that depend on it (SC-5). No evidence requirement in this plan presupposes an open decision: blocked items verify their unconditional invariant now and their mechanism-dependent portion after the decision closes (§5.3).

## 9. Reservations Gate

### 9.1 Gates before Reservations implementation

Reservations implementation — any implementation — waits behind these corrections. The gate is the correction itself, not its priority class.

| Gate | Corrections | Document 03 precondition | Why Reservations waits |
|---|---|---|---|
| Scope from identity | C-01 | P1 | Every reservations request would otherwise inherit client-selected scope |
| Isolation declared for the connecting role | C-02 | P2 | Without the database defence layer, reservations data is unprotected at depth (OD-02 blocks mechanism, not the gate) |
| Reproducible schema baseline | C-03 | P3 | Reservations schema work cannot be verified against a baseline that disagrees with itself |
| Cross-context use via exported surface | C-07, C-08 | P4 | Reservations is the most reached-into context (35 internal-import edges); the exported surface must be the only way in |
| Uniform endpoint authorization | C-14, C-15 | P7 | New reservations endpoints must inherit fail-closed authorization (OD-09 blocks vocabulary selection only) |
| Transactions on money paths | C-09 | P8 | Reservations interacts with deposit and folio money paths |
| Single authoritative guest and folio stores | C-12 | P9, P10 | Reservations must not add a second writer to guest identity or folio concepts |
| Executable evidence for behaviour changes | C-24 | P11 | Reservations behaviour changes need evidence gates that do not exist yet |

### 9.2 What this section does not do

This section designs no Reservations table, endpoint, state set, workflow, or implementation order; it grades no readiness and changes no verdict. Reservations remains excluded from the review's correction scope per Document 03 §13 and Document 04 §3.4.6; N-07 stands. Document 03's readiness verdicts and Document 04 §3.4's firewall are unchanged by this plan.

## 10. Out of Scope, Deferred & Explicit Non-Goals

### 10.1 Conditioned gaps — deferred, no correction asserted

| Gap | OD | Status in this plan |
|---|---|---|
| GAP-23 — reporting/analytics data path | OD-05 | No correction; whether the current trigger-coupled path violates §6.5 cannot be stated until the path is chosen |
| GAP-24 — tenancy naming and isolation-root expression | OD-01 | No correction; the double-parameter defect itself is already covered inside GAP-02 via C-02 |
| GAP-25 — single contract per surface | OD-04 | No correction; the doubled path it would advertise is already covered inside GAP-13 via C-16 |
| GAP-26 — cross-domain identifier format | OD-08 | No correction; the typing violation itself is already covered inside GAP-09 via C-06 |

Each entry re-enters correction planning only after its open decision closes and its Document 05 classification is revisited in a future step — not this one.

### 10.2 Open-decision-dependent portions of confirmed corrections

- **C-02 / OD-02**, **C-06 / OD-07 and OD-08**, **C-13 / OD-03**, **C-15 / OD-09** — survivor and mechanism selection only; invariants proceed (§5.3, SC-10).
- **C-24 / OD-10** — test strategy specifics (framework, coverage targets, suite organisation); the structural verification rules hold regardless.
- **OD-06 (process topology)** — affects nothing in this plan: single-process composition is a non-gap (N-02) and no correction changes topology.
- **Conditioned gaps GAP-23, GAP-24, GAP-25, GAP-26** — no items (§10.1).

### 10.3 Non-gaps and previously deferred items — no corrections

N-01 (sound architecture, preserved by §1.15), N-02 (single-process composition), N-03 (one physical database and existing schema count), N-04 (near-zero client surface), N-05 (no production imports from client code), N-08 (component size), and N-09 (unused library packages) are not gaps and receive no correction. N-06 (test strategy specifics, OD-10) and N-07 (Reservations internal design) stay deferred exactly as Document 05 classified them.

### 10.4 Explicit non-goals of this plan

- **No implementation.** No code, schema, migration, test, configuration, or CI change is made, instructed, or sketched; this step changed nothing but this file.
- **No big-bang, no speculative redesign.** No microservices split; no database-per-context mandate (OD-07 open); no event-driven rewrite; no process-topology change (OD-06 open); no framework or platform replacement.
- **No reopening.** No Document 01–05 content, classification, severity, readiness verdict, or residual is changed; no new `ARCH-` finding exists.
- **No decision-making.** No OD-01…OD-10 is resolved, pre-selected, or assumed; no conditioned gap is asserted.
- **No project management.** No dates, durations, estimates, owners, assignments, sprints, or task breakdowns appear anywhere in this document; §7 is dependency, never schedule.
- **No Reservations work.** No design, spec, readiness grading, or ordering for Reservations (§9.2).
- **No product work.** No feature changes, performance tuning, or dependency upgrades; where a correction's future execution may require a dependency decision, that decision belongs to the executing step, not here.

### 10.5 Final tally

| Tally | Count |
|---|---|
| Correction workstreams | 8 |
| Correction items (C-01…C-24, all Status PLANNED) | 24 |
| P0 / P1 / P2 / P3 / P4 | 4 / 10 / 8 / 2 / 0 |
| Items marked BLOCKED BY OPEN DECISION (C-02, C-06, C-13, C-15) | 4 |
| Confirmed gaps covered | 22 of 22 |
| Conditioned gaps deferred | 4 |
| Documents created or modified by this step | 1 (this file only) |

---

**End of Document 06 — Architecture Correction Plan.**

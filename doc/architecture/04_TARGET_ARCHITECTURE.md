# Document 04 — Target Architecture

**Step 04 of the XYLO Project-Wide Architecture Review**

Status: COMPLETE — ARCHITECTURAL DEFINITION ONLY
Follows: `docs/architecture/01_CURRENT_ARCHITECTURE_AUDIT.md`, `docs/architecture/02_OPEN_EVIDENCE_RESOLUTION.md`, `docs/architecture/03_ARCHITECTURE_FINDINGS_AND_VERDICT.md`
Date: 2026-10-08

---

## How to read this document

### Purpose

This document is an **independent architectural contract**. It defines where the XYLO system is intended to go, stated as normative rules and ownership boundaries. It is written to be used later by a Gap Analysis, by correction/refactoring planning, and by future domain implementation — none of which is performed here.

### Authoritative inputs

| Input | Role |
|---|---|
| `docs/architecture/01_CURRENT_ARCHITECTURE_AUDIT.md` | Current-state finding register, classification vocabulary, original verdict |
| `docs/architecture/02_OPEN_EVIDENCE_RESOLUTION.md` | Resolved evidence for the ten open current-state questions (E-01…E-10) |
| `docs/architecture/03_ARCHITECTURE_FINDINGS_AND_VERDICT.md` | **Authoritative current-state findings and verdict**: 53 findings (7 SOUND, 27 ARCHITECTURAL PROBLEM, 8 TECHNICAL DEBT, 6 LEGACY RESIDUE, 5 IMPLEMENTATION BUG, 0 UNCERTAIN); verdict `STRUCTURALLY PROBLEMATIC BUT REPAIRABLE`; repairability `MODERATE`; four global preconditions P1–P4; 7 domains `READY WITH ARCHITECTURAL CONSTRAINTS`, 5 domains `NOT READY`; Reservations excluded from readiness assessment |

The step instruction referred to these as `01_ARCHITECTURE_AUDIT.md` and `02_ARCHITECTURE_DECISIONS.md`; the repository's actual documents are the two named above, and those are the files that were read.

Documents 01–03 are **evidence and constraints about the current system**. They are not the target, and nothing in them is automatically converted into an implementation task by this document.

### The three things kept separate

| | Definition | Where it lives |
|---|---|---|
| **CURRENT STATE** | What exists today | Documents 01, 02, 03 |
| **TARGET STATE** | What the architecture must be | **This document (04)** |
| **REMEDIATION** | How the system gets from current to target | **Not produced. Out of scope for this step.** |

This document contains only the Current → Target definition. It contains no gap analysis, no remediation plan, no task breakdown, no sequence, no owners, and no estimates.

### Evidence classification legend

Every substantive target decision in this document carries one of three marks:

| Mark | Meaning |
|---|---|
| **[TP]** | **TARGET PRINCIPLE** — an architectural rule that holds independently of the current code; it would be true of XYLO even if the current implementation were different |
| **[EC]** | **EVIDENCE-CONSTRAINED TARGET** — a target decision shaped by a confirmed current-state constraint; the basis (a finding ID, a precondition, or an evidence item) is cited |
| **[OD]** | **OPEN / REQUIRES DECISION** — the contract is stated, but the decision cannot be safely finalised from the available evidence; recorded in Appendix A |

Marking is deliberate: manufactured certainty is treated as a defect of this document.

### What this document does not do

- It does **not** modify application code, Prisma schemas, migrations, the database, tests, or CI/CD.
- It does **not** create implementation tasks, a correction plan, a migration sequence, or a work breakdown.
- It does **not** perform a Gap Analysis or assign any current-state finding to future work.
- It does **not** reopen or amend Documents 01–03, and it creates **no new current-state findings** (`ARCH-` identifiers are cited as basis only; this document introduces no new ones).
- It does **not** design Reservations: no tables, endpoints, commands, state set, UI, or implementation.
- It does **not** redesign screens, workflows, or UX.
- It does **not** silently adopt an implementation technology or mechanism merely because it exists in the codebase today.

Rule identifiers introduced here are **architectural contract rules** (`AC-nn`) and **open decisions** (`OD-nn`). Neither is a finding identifier.

---

## 1. Architectural Principles

These are the non-negotiable principles governing the target system. Sections 2–16 elaborate them; where a later section appears to conflict with a principle, the principle governs.

**1.1 One authoritative owner for every business state.** **[TP]**
Every piece of business state has exactly one domain that may write it. Any number of domains may read it, but only through the owner's public contract. Two live writers of the same business fact do not exist in the target state.

**1.2 Identity-derived scope; client input selects, never authorises.** **[EC — P1, ARCH-007]**
Tenant and property scope is established from the authenticated principal. A client-supplied identifier may only choose among scopes the principal already holds. A client-controlled property identifier is never, by itself, authorization.

**1.3 Contracts are the only cross-domain surface.** **[EC — P4, ARCH-013]**
Domains interact exclusively through their declared public contracts. Internal layers — domain entities, events, ports, handlers, repositories, persistence — are private to their owning domain.

**1.4 The repository is the structural source of truth.** **[EC — P2, P3, ARCH-008, ARCH-010]**
Any object that shapes the database or the security posture of the data — tables, indexes, constraints, policies, functions, triggers — is declared in this repository and reproducible from it. An object that exists only in an environment is a defect of the target state.

**1.5 Fail closed at every trust boundary.** **[TP]**
Authentication, authorization, tenancy, and external integration boundaries deny by default. Absence of a requirement, of a decorator, of a header, or of a configuration value results in refusal, never in permissive fallback.

**1.6 Exactly one reachable mechanism per cross-cutting concern.** **[EC — ARCH-015, ARCH-016, ARCH-017]**
Authorization, command/query dispatch, transaction participation, idempotency, audit, exception translation, configuration, and HTTP client access each have exactly one reachable implementation in the target state. Two implementations that a module could choose between are not permitted.

**1.7 One business operation, one transaction boundary.** **[EC — ARCH-019, P8]**
A business operation that affects money, inventory, availability, or reservation state commits atomically or not at all. Partial commit of such an operation does not exist in the target state.

**1.8 The domain decides; transport carries.** **[EC — ARCH-014, ARCH-034]**
Business rules and invariants live in domain layers. Controllers, background jobs, webhooks, and UI components translate and orchestrate; they do not hold business rules.

**1.9 Capability honesty.** **[EC — ARCH-027, ARCH-028, ARCH-032, ARCH-038]**
A subsystem in the target state is either active and observable, or explicitly declared dormant. Simulated or fabricated output is labelled as such at the point it is produced and can never be presented as live business data.

**1.10 Boundaries are enforced by machines, not by convention.** **[EC — ARCH-031]**
Dependency direction, layer rules, and ownership rules are checked by automated architectural tests in the build. A rule that only exists in prose is not yet an architectural rule.

**1.11 External systems integrate at adapters only.** **[TP]**
CRS, OTA channels, payment providers, messaging providers, telephony, storage, and any future third party reach the system through a dedicated adapter that translates at the boundary. No external system reads or writes domain state directly.

**1.12 Observability is a platform contract in which every domain participates.** **[EC — ARCH-005]**
Logging, tracing, metrics, correlation, and audit are defined once at platform level and emitted by every domain against the same contract.

**1.13 Legacy is quarantined; growth happens in owned contexts.** **[EC — ARCH-041, ARCH-042, ARCH-043]**
Superseded code and data are contained and identifiable. New capability is added only inside an owned context, never inside a legacy structure.

**1.14 Defer the mechanism, fix the contract.** **[TP]**
Where the available evidence does not justify choosing an implementation mechanism, the target fixes the contract the mechanism must satisfy and records the mechanism as `OPEN / REQUIRES DECISION`. Silence is not permitted; neither is a choice justified only by what already exists.

**1.15 Preserve what is already sound.** **[EC — ARCH-001, ARCH-002, ARCH-003, ARCH-004, ARCH-006, ARCH-053]**
The target extends the properties the current-state audit found sound — clean layer direction and a single composition root, the DDD domain cores, command/query dispatch with validation and transaction pipes, the transactional availability assertion engine with its database-backed test harness, verified OTA ingress, and fail-closed authorization ordering. The target does not require re-inventing them.

**1.16 The target is stated independently of the accident.** **[TP]**
Where current structure and sound architecture disagree, this document states the sound architecture. Current shape is cited only as basis (`[EC]`), never as justification by inertia.

---

## 2. Domain Boundaries

### 2.1 Context map

The target is a set of **bounded contexts** grouped into four strata. The strata express dependency allowance (§4), not deployment units: the target runs as a single system composed of these contexts.

| Stratum | Contexts | May be depended on by |
|---|---|---|
| **T0 — Platform & foundation** | Identity & Access · Tenant Registry & Configuration · Workspace | all strata |
| **T1 — Core commercial** | Guest Profile · Reservations · Availability · Rates & Pricing · Groups & Allotment · Channels & Distribution · Activities & Events | T1 (per §2.2–2.5 rules), T2 |
| **T2 — Operational & financial** | Front Office · Housekeeping · Billing · Inventory · Purchasing | T2 (per rules), applications |
| **T3 — Clients** | Web application · Admin application · Mobile application | none (they consume contracts) |

Supporting capabilities that own **no** business state are listed in §2.6.

### 2.2 Platform & foundation contexts (T0)

| Domain | Owns | Does NOT own | Authoritative business state | Public contract | Allowed dependencies |
|---|---|---|---|---|---|
| **Identity & Access** | Authentication principals, credential verification, sessions and tokens, role catalogue, permission catalogue | Any business transaction data; property master; tenant scope policy itself | subjects, roles, permissions, sessions | authenticate · validate principal · list permissions · check permission | platform infrastructure only |
| **Tenant Registry & Configuration** | Property/hotel master, organisation units, global reference data (currency, country, locale), declarative configuration (settings, feature configuration, tax configuration as reference data) | Business transactions; authorization decisions; workspace layout; any domain's transactional records | properties, org units, reference data, configuration records | read property · read reference data · read configuration | platform infrastructure only |
| **Workspace** | Operator workspace layout, tabs, widget placement, dashboard preferences — tenant-scoped presentation configuration | Business data; widget *content* (produced by the owning business domain) | workspace, tab, widget configuration | read/write own workspace configuration | Identity & Access, Tenant Registry |

### 2.3 Core commercial contexts (T1)

| Domain | Owns | Does NOT own | Authoritative business state | Public contract | Allowed dependencies |
|---|---|---|---|---|---|
| **Guest Profile** | Person identity and contact data, profile linkage, guest preferences, loyalty membership | Reservation participation; stay records; folio; any transactional state | profiles, identity/contact attributes, profile links, preferences | lookup · create · update profile · resolve party | Tenant Registry |
| **Reservations** | The reservation aggregate and its lifecycle, guest *participation references* (who is on which reservation, in what role), requested stay intent (dates, room types, quantities), holds and waitlist | Sellable capacity; rate definitions; folio; room assignment; room status; person identity | reservations, participation references, lifecycle history | see §3.4 — create / amend / confirm / guarantee / cancel / reinstate / no-show / read · publish reservation facts | Availability, Rates & Pricing, Guest Profile, Tenant Registry |
| **Availability** | Sellable capacity per property and space-type per date, the assertion ledger for that capacity, restrictions (arrival/departure windows, length-of-stay, stop-sale, sell limits), oversell and limit policy state | Pricing; reservations; block contracts; channel state; any capacity state of another domain | capacity, assertion balances and movements, restrictions | assert · release · read capacity · apply restriction | Tenant Registry |
| **Rates & Pricing** | Rate plans, seasons and components, quotations and their outcomes | Capacity; reservations; folio balances; charges actually posted | rate definitions, quotations | quote · read rate definitions | Availability, Tenant Registry |
| **Groups & Allotment** | Group and allotment contracts, allotment quotas, the pickup ledger, block lifecycle | Sellable capacity (Availability); reservations (Reservations); folio (Billing) | blocks, contracts, quotas, pickup records | create / modify / release block · allocate capacity · record pickup · read group | Availability, Reservations, Tenant Registry |
| **Channels & Distribution** | Channel configuration and mapping, inbound normalization and deduplication state, outbound delivery state, channel synchronisation status | Capacity; rates; reservations; any authoritative commercial state of another domain | channel configuration, mapping, delivery and sync records | request distribution inputs from owning contracts · ingest normalized inbound · read delivery state | Availability, Rates & Pricing, Reservations, Tenant Registry |
| **Activities & Events** | Banquet, venue, and event records and their lifecycle | Space capacity (Availability); folio (Billing); rate definitions (Rates) | events, banquet bookings, venue reservations | create / modify / read event | Availability, Rates & Pricing, Billing, Tenant Registry |

### 2.4 Operational & financial contexts (T2)

| Domain | Owns | Does NOT own | Authoritative business state | Public contract | Allowed dependencies |
|---|---|---|---|---|---|
| **Front Office** | Stay execution: arrival and departure execution records, the room-assignment binding (which room a stay occupies), transfer/upgrade execution records, front-desk operational queues and desk-level operational items | Reservation lifecycle; folio; room cleanliness; sellable capacity; person identity | room assignments, arrival/departure execution records, desk queues | check in · check out · assign room · transfer / upgrade · read operational queue | Reservations, Housekeeping (readiness), Billing, Guest Profile, Availability (read), Tenant Registry, Identity & Access |
| **Housekeeping** | Room cleanliness/inspection status, out-of-order state, housekeeping tasks, lost & found, and front-office/housekeeping reconciliation records | Room master; occupancy; reservation; folio; capacity | room status, tasks, inspection results, discrepancy records | read / update room status · manage tasks · record reconciliation discrepancy | Tenant Registry (room master), Front Office (published occupancy facts) |
| **Inventory** | Item master, warehouses, stock ledger and balances, valuation and inventory periods | Purchase lifecycle; vendor relationships; consumption decisions that do not move stock | items, warehouses, stock movements, stock balances, periods | receive · adjust · consume stock · read stock | Tenant Registry |
| **Purchasing** | Vendor master, purchase requisitions, approval records, purchase orders, receipt records — the procurement lifecycle | Stock levels; item master; folio | vendors, requisitions, approvals, purchase orders, receipts | raise · approve · order · receive · read | Inventory, Tenant Registry |
| **Billing** | Folios, postings, charges, balances, settlement and payment-attempt records, the posting/routing code catalogue, posting-level currency conversion, and the payment-provider integration boundary | Reservation price agreement; room state; stock; person identity | folios, postings, balances, settlements, payment attempts | open folio · post / transfer / reverse · settle · close · read balance | Tenant Registry, Identity & Access (actor) |

### 2.5 Clients (T3)

| Domain | Owns | Does NOT own | Authoritative business state | Public contract | Allowed dependencies |
|---|---|---|---|---|---|
| **Web application** | Operator-facing presentation structure and client-side interaction state | Any business state; any authoritative computation | none (client-side caches are derived, not authoritative) | consumes API contracts | API contracts of any context |
| **Admin application** | Corporate presentation structure | Any business state | none | consumes API contracts | API contracts of any context |
| **Mobile application** | Guest/staff presentation structure | Any business state | none | consumes API contracts | API contracts of any context |

All three clients consume the **same** contract surface; difference in reach is expressed through authorization scope, not through a separate API. **[EC — ARCH-036]**

### 2.6 Supporting capabilities (own no business state)

| Capability | Responsibility | Explicitly does NOT own |
|---|---|---|
| **Telemetry & Audit** | Logging, tracing, metrics, correlation identifiers, the access/audit pipeline | Domain state-change facts, which are recorded transactionally by the owning domain (§12) |
| **Notifications** | Delivery adapters (email, push, SMS) and delivery status | Deciding *what* and *when* to notify — that belongs to the notifying domain |
| **Reporting & Analytics** | Derived read models and exports | Any authoritative business state (§6.5) — **[OD]** see OD-05 |
| **Workflow / Job Runtime** | Execution of scheduled and queued work | Which work exists — each job belongs to exactly one owning domain (§11.5) |
| **Storage & Delivery** | Object storage access | Business content ownership |
| **Edge / API surface** | Route prefixing, TLS termination, rate limiting, principal extraction | Authorization decisions, which belong to Identity & Access and the route contract |

Payment provider adapters sit **behind Billing's contract** and are Billing's integration boundary; they are not a free-standing capability. **[EC — ARCH-038]**

### 2.7 Boundary rules

1. The context map above is **normative**: a capability whose subject already belongs to a listed context MUST be placed in that context. Creating a second context for the same subject is forbidden. **[TP]**
2. A context is defined by **business ownership**, never by technical layer. "The reporting module", "the repository layer", and "the services layer" are not contexts. **[TP]**
3. A context MUST have, and publish, exactly one public contract surface (§4.1). A context with no published contract is not integrated and MUST be declared dormant (§14). **[EC — ARCH-027, ARCH-028]**
4. Placement of a *new* capability is decided by which context owns the state it reads or writes — not by which module is nearest. **[TP]**
5. Contexts listed in §2.6 own no business state and therefore MUST NOT acquire any; a supporting capability that begins to own business state becomes a context and must appear in §2.2–2.4. **[TP]**

---

## 3. Domain Ownership

### 3.1 The single-owner rule

1. For every business concept and every lifecycle state, **exactly one** context is the authoritative owner and the only writer. **[TP]**
2. All other contexts read through the owner's public contract, or through a derived read model that is explicitly non-authoritative (§6.5). **[TP]**
3. A context MAY record *its own* fact that references another context's state (for example, "reservation X was executed as an arrival at T"), provided it records its own observation and never a copy of the other context's state value. **[TP]**
4. Time-frozen snapshots of another context's data are permitted **only** where the snapshot is explicitly historical, is never updated after creation, and is never used as a live value. Live copies of another context's state are forbidden. **[EC — ARCH-011, ARCH-012]**

### 3.2 Ownership of key business concepts

| Business concept / state | Authoritative owner | Readers (via contract or published facts) | MUST NOT write it |
|---|---|---|---|
| **Reservation and its lifecycle** | **Reservations** (§3.4) | Front Office, Channels & Distribution, Groups & Allotment, Reporting | every other context |
| **Guest/person identity and contact data** | **Guest Profile** | Reservations, Front Office, Billing, Reporting | every other context — **[EC — ARCH-012, P9]** |
| **Reservation participation (who is on which reservation)** | **Reservations** | Front Office, Reporting | every other context |
| **Sellable capacity per date** | **Availability** | Reservations, Groups & Allotment, Channels & Distribution, Rates & Pricing, Front Office, Activities & Events, Reporting | every other context — **[EC — Document 01 §6.3: ≥6 counter locations]** |
| **Restrictions / stop-sale / sell limits** | **Availability** | Reservations, Channels & Distribution, Reporting | every other context |
| **Rate definitions and quotations** | **Rates & Pricing** | Reservations, Channels & Distribution, Activities & Events, Reporting | every other context |
| **Group blocks, quotas, pickup** | **Groups & Allotment** | Availability (capacity effects applied through Availability's contract), Reservations, Reporting | every other context |
| **Stay execution (arrival, departure, room assignment, transfer/upgrade)** | **Front Office** | Housekeeping, Billing, Reporting | every other context — including Reservations, which requests execution through Front Office's contract |
| **Room cleanliness / inspection / out-of-order status** | **Housekeeping** | Front Office, Reporting | every other context |
| **Room master (room number, type, floor)** | **Tenant Registry** | Availability, Front Office, Housekeeping, Reporting | every other context |
| **Folio, postings, balances, settlement** | **Billing** | Front Office, Activities & Events, Reporting | every other context — **[EC — ARCH-025, P10]** |
| **Item master, stock levels, stock movements** | **Inventory** | Purchasing, Housekeeping, Reporting | every other context — **[EC — ARCH-011, P6]** |
| **Vendors, purchase requisitions, orders, receipts** | **Purchasing** | Inventory (receipt effects applied through Inventory's contract), Reporting | every other context — **[EC — ARCH-011]** |
| **Property master, reference data, configuration** | **Tenant Registry** | every context (read) | every other context — **[EC — ARCH-026]** |
| **Subjects, roles, permissions, sessions** | **Identity & Access** | every context (authorization evaluation) | every other context |
| **Event/banquet records** | **Activities & Events** | Billing, Availability (capacity effects via contract), Reporting | every other context |
| **Channel configuration, mapping, delivery state** | **Channels & Distribution** | Reporting, operations | every other context |
| **Derived read models and exports** | **Reporting & Analytics** | clients | every business context (no business context writes another's read model) |

### 3.3 Lifecycle-state ownership

A lifecycle is a chain of states owned end-to-end by one context. The target assigns lifecycles as follows; the individual states within each lifecycle are a decision for that domain's own specification and are deliberately not enumerated here.

| Lifecycle | Owner | Cross-boundary transitions |
|---|---|---|
| Reservation lifecycle | **Reservations** | A check-in or check-out *operation* is performed by Front Office; the resulting reservation state transition is requested by Front Office **through Reservations' contract** — Front Office never writes reservation state |
| Stay execution lifecycle (arrival → in-house → departed) | **Front Office** | Published as facts for Housekeeping, Billing, Reporting |
| Room status lifecycle (dirty → clean → inspected / out-of-order) | **Housekeeping** | Read by Front Office before assignment |
| Folio lifecycle (open → active → settled → closed) | **Billing** | Folio opening and posting are requested by the operating context through Billing's contract |
| Stock movement lifecycle | **Inventory** | Purchase receipts are recorded by Purchasing and applied to stock through Inventory's contract |
| Procurement lifecycle (requisition → approval → order → receipt) | **Purchasing** | Stock effects applied through Inventory's contract |
| Block lifecycle (draft → contracted → allocated → released/consumed) | **Groups & Allotment** | Capacity effects applied through Availability's contract; reservations created through Reservations' contract |
| Channel synchronisation lifecycle | **Channels & Distribution** | Inputs gathered from Availability, Rates & Pricing, and Reservations contracts |
| Event/banquet lifecycle | **Activities & Events** | Space capacity effects applied through Availability's contract; charges through Billing's contract |

**Rule:** a cross-boundary transition MUST be expressed as a command to the owning context's contract or as an accepted published fact. It MUST NOT be expressed as a direct write to the owner's state. **[TP]**

### 3.4 The Reservations boundary

Reservations is strategically important and is deliberately specified here only as a **boundary**, not as a design.

1. **Reservations owns its reservation lifecycle state.** No other context creates, amends, cancels, or transitions reservation state except by issuing a command to Reservations' public contract. **[EC — P4, ARCH-013]**
2. **Reservations MUST expose a controlled public contract**: the command surface for lifecycle transitions, the query surface for reading reservations, and the published facts other contexts consume. That contract is the entire Reservations interface. **[TP]**
3. **Other contexts MUST NOT depend on Reservations' internal layers** — aggregates, entities, value objects, domain events as classes, ports, handlers, repositories, or persistence — and MUST NOT read Reservations' tables. Import depth into Reservations beyond its published contract is forbidden regardless of how convenient it is. **[EC — P4, ARCH-013: 35 internal edges across 7 layers today]**
4. **Reservations depends on** Availability (capacity assertion), Rates & Pricing (quotation), Guest Profile (identity), and Tenant Registry — through their contracts. It depends on no operational context. **[TP]**
5. **Front Office is Reservations' principal consumer** and MUST consume only the contract. Conversely, Reservations MUST NOT reach into Front Office's internals. **[TP]**
6. **Out of scope here:** Reservations' tables, endpoints, commands, state set, aggregate design, UI, and implementation sequence. Those belong to the Reservations domain specification, not to this document. Nothing in §2–§16 should be read as designing them.

---

## 4. Dependency Direction

### 4.1 The layer model

Every context is internally structured in these layers. The table is normative: a layer may depend only on layers to the right of the "May depend on" column entries listed.

| Layer | Contains | May depend on | MUST NOT depend on |
|---|---|---|---|
| **Public contract** | Exported commands, queries, DTOs, result and error types, published fact types | its own context's application/domain types; shared kernel value types | any other context's internals; its own context's infrastructure or persistence; another context's persistence |
| **Application layer** | Use-case handlers, orchestration, transaction boundaries, authorization checkpoints, ports (interfaces) | its own contract, its own domain layer, platform services | another context's application/domain layers; another context's persistence; any UI; external SDKs directly |
| **Domain layer** | Aggregates, entities, value objects, domain services, invariants, domain error definitions | shared kernel pure types only | any framework, ORM, transport, UI, another context, file system, clock/network without abstraction |
| **Infrastructure layer** | Implementations of the context's own ports (persistence, caches, external clients, bus adapters) | its own application ports; platform infrastructure | another context's modules or ports |
| **Persistence** | The context's schema/client, repositories, query objects | **its own schema only** | any other context's schema or tables |
| **Integration adapters** | External-system clients, webhook verification, provider SDKs, normalization | its own application ports | any context's domain internals; any database directly (except its own schema) |
| **Presentation (HTTP)** | Controllers, route definitions, request/response mapping, validation at the edge | its own application/contract | persistence, ORM, another context's internals, business rules (§1.8) |

### 4.2 Allowed dependencies

1. **Downward within a context** (presentation → application → domain; infrastructure implements application ports). **[TP]**
2. **Sideways only through contracts**: context A → context B's **public contract** exclusively. **[EC — P4, ARCH-013]**
3. **Any context → T0 platform** (identity checks, registries, configuration, telemetry). **[TP]**
4. **T0 platform → no business context.** Platform MUST NOT depend on any T1/T2 context. **[EC — Document 01: `platform → modules` = 0 edges, classified SOUND, ARCH-001]** **[TP]**
5. **Reporting & Analytics → contracts and published facts of multiple contexts**, read-only. **[TP]**
6. **Clients → API contracts only** (§10). **[EC — Document 01 §8.5: 0 production imports of API or database source — a property to preserve]** **[TP]**
7. **Type-only import of another context's published fact types** is permitted for consumption purposes and does not count as a dependency edge, provided it performs no registration, construction, or instantiation in the importing context (§4.4). **[TP]**

### 4.3 Forbidden dependencies

The following are forbidden in the target state, without exception:

1. One context's **domain or application layer** imported by another context. **[EC — ARCH-013]**
2. One context's **persistence** accessed by another context — by ORM client, raw SQL, view, or convenience helper. **[EC — ARCH-014, ARCH-025]**
3. **Presentation → persistence** (a controller or route handler that touches a database client). **[EC — ARCH-014]**
4. **Infrastructure → application modules of another context** (any reverse edge into `modules`). **[EC — ARCH-030]**
5. **Any dependency from a business context into platform that would make platform depend on business state.** **[TP]**
6. **Client-side code importing backend source or database clients** in any form. **[TP]**
7. **External-system SDK usage outside an integration adapter.** **[TP]**
8. **A second implementation of a cross-cutting concern reachable from application code** (§1.6). **[EC — ARCH-015, ARCH-016, ARCH-017]**
9. **Mutual *synchronous* dependence between two contexts** (§4.4). **[EC — ARCH-029]**

### 4.4 The cycle rule

1. Two contexts MUST NOT depend on each other synchronously. Where two contexts need information from each other, **exactly one direction MUST be synchronous (contract invocation) and the other MUST be asynchronous (consumption of published facts)**. **[EC — ARCH-029: Availability⇄Rates forward-reference cycle today]**
2. Published fact types are defined and owned by the **publisher**. A consumer importing those types for deserialization and handling is an asynchronous edge, not a synchronous one. **[TP]**
3. An unbalanced one-way contract reference (context A designed expecting context B to depend back, when it does not) MUST NOT exist. **[EC — ARCH-029]**
4. Registration-time cycles — contexts whose module composition requires forward references to boot — are forbidden outright; composition MUST resolve by strata order (T0 → T1 → T2 → T3). **[EC — ARCH-029]**

## 5. Cross-Domain Communication

Contexts never reach into each other. They exchange exactly two things: **invocations of a published contract** (synchronous) and **published facts** (asynchronous). This section is the contract; the mechanism that carries either of them is open (5.7).

### 5.1 The communication contract

1. A context may depend on another context only through that context's public contract (4.2). **[TP]**
2. Two forms of cross-context communication exist and no third form is permitted: contract invocation and fact publication. **[TP]**
3. Direct reads of another context's tables, repositories, or database client — and any cross-context raw SQL — are not communication; they are boundary violations (4.3). **[EC — ARCH-014: 233 module files reach `PrismaService`, 144 use raw SQL, 6 controllers inject it directly]**
4. Shared mutable in-process state (a registry, a singleton cache, a "shared" module holding business data) is not communication either. **[TP]**

### 5.2 Synchronous contract invocation

1. Exactly one direction between any two contexts may be synchronous; the reverse direction, if one exists, is asynchronous (4.4). **[EC — ARCH-029: one real `forwardRef` cycle (Availability-RatesInventory), one latent cycle masked by an unregistered module, one unbalanced forward reference]**
2. A synchronous call crosses exactly one boundary: from the caller's application layer into the callee's public contract. It never crosses into the callee's application, domain, infrastructure, or persistence layers. **[EC — ARCH-013: front-office reaches seven internal layers of reservations across 35 of the 78 cross-module internal-import edges]**
3. The call carries typed inputs and returns typed results or typed failures. No context reads another context's output by inspecting its data. **[TP]**
4. A synchronous contract call MUST NOT be used where the caller only needs to observe something that happened; published facts exist for that (5.3). **[TP]**
5. Cross-context synchronous calls are in-process by default. Whether any pair of contexts is deployed as separate processes is open (5.7, OD-06); the contract itself is transport-independent (9.7). **[OD]**

### 5.3 Published facts (events)

1. A fact is an immutable, past-tense statement that has happened in the publishing context (`ReservationConfirmed`), typed and owned by the publisher. **[TP]**
2. Facts are not commands. A fact MUST NOT be routed back into another context as an instruction to write, and a consumer MUST NOT treat receipt of a fact as permission to bypass its own contract or invariants. **[TP]**
3. Delivery is at-least-once; consumers are idempotent and key on a stable fact identity. **[TP]**
4. Ordering is guaranteed only within one aggregate/root instance; consumers MUST NOT assume global ordering. **[TP]**
5. A publisher MUST NOT inspect, depend on, or enumerate its consumers. **[TP]**
6. Fact publication happens inside the owning context's transaction boundary (as part of the commit or as an outbox record committed with it), so a fact never announces a state that may be rolled back. Exactly one publication path exists per context, and that path demonstrably carries facts. **[EC — ARCH-018: in-process dispatcher has zero handler registrations and always no-ops, a second outbox processor has no callers, the analytics consumer is unregistered — only the `events` leg carries traffic (Document 02 E-09)]**

### 5.4 Integration messages are not internal facts

1. Payloads arriving from CRS, OTA, webhooks, or channel partners are integration messages: parsed, validated, and verified inside an integration adapter, then expressed as contract invocations into the owning context. **[EC — ARCH-006: OTA ingress live with HMAC verification, dedupe, and creation through the reservations facade — sound, preserve]**
2. An external payload MUST NOT be re-published internally as if it were a native fact, and MUST NOT fan out to internal contexts without first passing through the owning context's contract. **[TP]**
3. The reverse also holds: internal facts MUST NOT be exported verbatim to external systems; outbound adapters translate (11.4). **[TP]**

### 5.5 What counts as communication

| Mechanism | Status | Basis |
|---|---|---|
| Public contract invocation | REQUIRED — the synchronous form | **[TP]** |
| Published facts | REQUIRED — the asynchronous form | **[TP]** |
| Direct table/repository access across contexts | FORBIDDEN | **[EC — ARCH-014]** |
| Cross-context raw SQL | FORBIDDEN | **[EC — ARCH-014]** |
| Import of another context's internal layers | FORBIDDEN | **[EC — ARCH-013, Document 03 P4]** |
| Shared module holding business state | FORBIDDEN | **[EC — ARCH-030]** |
| HTTP call between backend contexts | Permitted only as a transport for a published contract; whether it occurs at all depends on deployment topology | **[OD — see OD-06]** |
| External payload straight to internal tables | FORBIDDEN | **[EC — ARCH-006]** |
| Parallel second mechanism for the same purpose | FORBIDDEN | **[TP]** |

### 5.6 Failure semantics

1. A synchronous failure is a typed result the caller must handle. It MUST NOT be swallowed, logged-and-ignored, or converted into silent success. **[TP]**
2. An asynchronous failure enters retry under a bounded policy and, on exhaustion, lands in a dead-letter or failed state that is observable and attributable to the owning context. A fact that cannot be delivered MUST NOT disappear. **[EC — ARCH-018, ARCH-032: queues nothing listens on; an outbox processor with no callers]**
3. Every communication path has exactly one owner responsible for its health. Paths with no listener, no caller, or no registrant do not exist in the target. **[EC — ARCH-018, ARCH-032, ARCH-042: 82 files have zero non-test importers]**

### 5.7 Mechanism selection (open)

The contract above is fixed; what carries it is not decided here.

1. Whether cross-context calls are in-process only, or whether some contexts are separate processes — open as **OD-06**. **[OD]**
2. Which mechanism carries facts and background jobs (in-process bus, message broker, durable workflow engine, or a combination) — open as **OD-03**. The evidence cannot choose: an in-process dispatcher with zero registrations, a second outbox processor with no callers, an unregistered analytics consumer, three overlapping Temporal layers whose single workflow was never registered, queues nothing listens on, and only the BullMQ `events` leg carrying traffic (Document 02 E-09). **[OD]**
3. Whichever mechanism is selected, 5.1-5.6 apply unchanged, and exactly one mechanism family exists per purpose — no parallel stacks (1.6). **[TP]**

## 6. Data Ownership and Persistence

### 6.1 One authoritative store per concept

1. Every business concept has exactly one owning context and exactly one authoritative store. All other contexts hold references or derived copies, never competing truth. **[TP]**
2. Where the current system carries two or three stores for one concept, the target state is one store. Which store survives for each duplicated concept is governed by Document 03 preconditions P6, P9, and P10; this document states only the invariant, not the survivor. **[EC — ARCH-011, ARCH-012, ARCH-025, ARCH-026]**
3. Two writes to one concept through two clients or two DTO sets MUST NOT exist. **[EC — ARCH-026: two owners for property and two for company, on different Prisma clients with duplicate DTOs]**

### 6.2 Schema ownership and persistence boundaries

1. Each schema in the database belongs to exactly one context. Another context accessing it by any path other than contract invocation or an approved read model (6.5) is a boundary violation. **[EC — ARCH-009: 1,018 models over three schemas with a single migration owner; ARCH-014]**
2. Data access is defined and owned by the context that owns the data. There is no shared repository layer that other modules call into. **[EC — ARCH-014: the documented shared repository layer has 2 references while 233 module files reach `PrismaService` directly]**
3. A context's persistence detail — client, query shape, index, physical layout — is not part of its public contract and never appears in it. **[TP]**
4. Application code reaches data only through its own context's persistence layer. Controllers and route handlers do not hold a database client. **[EC — ARCH-014: 6 controllers inject `PrismaService` directly]**

### 6.3 Transaction ownership

1. The context that writes owns the transaction. One application operation = one transaction = one commit. **[TP]**
2. Multi-step writes that affect money or inventory complete inside a single transaction boundary. A path that cannot achieve this is expressed as an explicit sequence of contract invocations and facts — never as several independent commits that only appear atomic. **[EC — ARCH-019: check-in and check-out post money-affecting SQL outside the command transaction; four services perform multi-step writes with no transaction at all — Document 03 P8]**
3. No distributed transactions across contexts. Cross-context atomicity comes from facts and compensations, never from shared transactions or two-phase coordination. **[TP]**
4. A caller MUST NOT open a transaction around a callee's write, nor commit on the callee's behalf; transaction scope never leaks across a contract boundary. **[TP]**

### 6.4 Historical and append-only records

1. Ledger-style records — folio postings, charges, audit history — are append-only. Correction happens by a new entry that references the original, never by updating or deleting history. **[EC — ARCH-044: `folios`, `folio_postings`, `currencies`, `fx_rates` sit outside the ORM; ARCH-019]**
2. Point-in-time records captured when a decision was made (rate applied, availability committed) remain authoritative as history even after the live record changes. **[TP]**
3. Audit and operational history are written through one platform-wide mechanism (principle 1.11), not by per-context ad-hoc writers. **[EC — ARCH-017: two audit systems]**

### 6.5 Non-authoritative read models and projections

1. Derived data — dashboards, cross-context lists, search indexes, exports, analytics — lives in read models that are explicitly non-authoritative: rebuildable, disposable, and never the source of a business decision. **[TP]**
2. A read model is written by exactly one producer (its owning context or its approved projection path) and is never written by other contexts. **[TP]**
3. Anything a business rule must trust is authoritative data reached through its owner's contract (2.7, 6.1) — never a read model. **[TP]**
4. Where reporting today reads authoritative tables directly, the target's reporting and analytics path — read contracts versus rebuildable projections — is open as **OD-05**. **[OD]**

### 6.6 Legacy and abandoned data

1. Tables, columns, and triggers left behind by superseded functionality are not authoritative stores, and new behaviour MUST NOT be built on them. **[EC — ARCH-041, ARCH-052: two database-resident writers absent from every repository file, sitting on tables the application has abandoned]**
2. A legacy pipeline writing alongside a current pipeline for the same concept is dual truth: exactly one pipeline is authoritative (6.1), and no contract may target the other. **[EC — ARCH-043: the legacy `inventory_*` pipeline is still written by purchasing, procurement, and housekeeping alongside `xylo_inventory`, with no bridge and no reconciliation]**
3. Retention and removal of these tables and pipelines follow the legacy policy (14); this section states only that they are not a second store any contract may use. **[TP]**

### 6.7 Cross-domain references and identifiers

1. One context references another context's data by identifier from the contract — never by joining its tables or sharing its ORM entities. **[TP]**
2. Cross-schema keys are typed, not untyped strings. **[EC — ARCH-009: cross-schema keys remain untyped strings]**
3. The identifier convention across domains — format, directionality, stability — is open as **OD-08**, the repository mixing UUID, text, and composite primary keys. **[OD]**

### 6.8 Schema change execution

1. The only way application-owned schema reaches a database is through the migration pipeline (8). Application code MUST NOT create or alter schema at boot or at runtime. **[EC — ARCH-020: two services execute `CREATE TABLE`/`CREATE INDEX` during application boot]**
2. Database-resident behaviour — policies, triggers, functions — is migration-owned on the same rule (8.5). **[EC — ARCH-008, ARCH-041]**

## 7. Multi-tenancy / Property Isolation

### 7.1 Scope is identity-derived

1. For every non-public request, tenant and property scope is derived from the authenticated identity: verified claims and grants, resolved server-side. **[EC — Document 03 P1, ARCH-007]**
2. A client-controlled property identifier — header, query parameter, route segment, body field — **is not, and cannot become, authorization**. It may be accepted only as a request hint, and only after identity-derived scope has been established; the hint may narrow a request within the principal's scope or be rejected. It can never widen it. **[EC — ARCH-007: tenant scope comes from a client-controllable header with property authorization disabled]**
3. An endpoint with no authenticated principal carries no scope and therefore cannot reach tenant-scoped data or perform tenant-scoped mutation. Fail closed. **[TP]**
4. Scope is resolved once, at the boundary, from identity. Downstream code does not re-derive it from raw request data. **[TP]**

### 7.2 Authorization and scope are two different questions

1. "May this principal act?" is answered by authorization (9.6). "Which rows may they see?" is answered by identity-derived scope. Both must pass; neither substitutes for the other. **[EC — ARCH-007, ARCH-016]**
2. Scope narrowing is monotone: an operation may reduce the scope it acts within, never expand it. **[TP]**
3. Authentication precedes authorization and the sequence fails closed if any step is absent. This property holds today and is preserved. **[EC — ARCH-053: resolved safely, authentication precedes all authorisation, guards fail closed]**

### 7.3 The request scope context

1. A single request-scoped context object carries tenant, property, principal, and correlation identity. It is populated once at the boundary and read everywhere below. **[TP]**
2. Application, domain, infrastructure, persistence, and integration layers read that context. They MUST NOT read headers, cookies, or environment variables to discover scope. **[EC — ARCH-007]**
3. Background jobs, queue consumers, and workflow activities receive scope as an explicit, persisted input of the job. A job without scope does not run. **[TP]**
4. Exactly one scope-context type exists in the codebase; a second competing type is forbidden, in line with the one-mechanism rule (1.6). **[EC — ARCH-017, ARCH-015]**

### 7.4 Two enforcement layers

1. Application-level enforcement (guards and policy checks) is the primary authorization boundary: it decides what a request may do. **[EC — ARCH-016: the sequence is safe and fail-closed]**
2. Database-level isolation is a required second layer, not optional hardening. It exists so that a missed application check yields no cross-scope data rather than a breach. Its absence must be an explicit, reviewed property of a deployment — never an accident. **[EC — Document 03 P2, ARCH-008, ARCH-051 absorbed into ARCH-008]**
3. The two layers are independent in configuration: disabling or misconfiguring one MUST NOT silently disable the other. **[EC — ARCH-008: setters write one scope parameter while policies read another; the connecting role is superuser and bypass-capable; zero tables force policy enforcement]**
4. Neither layer may be "declared but not deployed": what the repository declares is what the database enforces (8.1). **[EC — ARCH-008: 386 tables and 392 policies unreproducible from source; the repository holds one policy file and it is absent live]**

### 7.5 The database-level isolation contract

The mechanism is open (OD-02); the contract is not. Whichever mechanism is chosen:

1. It is declared in version control, in this repository, alongside the schema it protects. **[EC — Document 03 P2, ARCH-008]**
2. It applies to the exact database role and connection path the application uses. A role that can bypass the mechanism — owner, superuser, bypass privilege — MUST NOT be the application's connection role, unless that fact is an explicit, reviewed exception recorded with the deployment. **[EC — ARCH-008: the role is owner plus superuser plus bypass]**
3. It is enforced by the storage engine rather than merely declared: enforcement is enabled on the tables it covers, not optional per session. **[EC — ARCH-008: zero tables force policy enforcement]**
4. The scope parameter is set and read by the same code path, with the same name and the same value, on every connection before any scoped statement executes. Setters and readers disagreeing on the parameter name is precisely the failure this contract forbids. **[EC — ARCH-008: setters write `app.current_tenant` while policies read `app.hotel_id`, so one of the two never takes effect]**
5. Isolation holds on every path to the database: application queries, background jobs, administrative scripts, and ad-hoc access alike. **[TP]**
6. Candidate mechanisms and their trade-offs are recorded as **OD-02**. No mechanism is adopted by default merely because one exists today in the live database. **[OD]**

### 7.6 Trust boundaries

| Boundary | What may cross | Rule |
|---|---|---|
| Public / unauthenticated | Nothing tenant-scoped | Public endpoints serve no tenant data and mutate nothing (7.1.3) — **[TP]** |
| Authenticated principal | Contract invocations within identity-derived scope | Scope from identity; a hint never widens it (7.1) — **[EC — ARCH-007]** |
| Backend context to backend context | Public contracts and published facts only | Section 5 — **[EC — ARCH-013]** |
| External system to backend | Verified integration messages | Verified before parse, adapter-owned (11.2) — **[EC — ARCH-006]** |
| Database session | Parameterised, scope-bound statements | Isolation contract (7.5) — **[EC — ARCH-008]** |
| Client application | Published contract types only | No backend source or database clients (4.3.6, 10.3) — **[TP]** |

### 7.7 Tenancy hierarchy (open)

1. The target requires exactly one isolation root: the single scope key expressed by every row-level policy, every guard, and every scoped query. **[TP]**
2. Whether that root is a tenant above properties, or the property itself — and how properties, hotels, and corporate accounts nest beneath it — is open as **OD-01**. **[OD]**
3. Until OD-01 is decided: two competing scope parameters already exist (`app.current_tenant` in setters, `app.hotel_id` in policies), and this contract forbids more than one scope parameter being authoritative. Mixed `tenant_id` and `hotel_id` column usage is treated as unresolved naming for one root, not as evidence of two roots. **[EC — ARCH-008]**
4. Cross-property visibility for a chain or corporate principal is a grant resolved from identity — not a query issued with scope removed. **[TP]**

### 7.8 Cross-property operations

1. Any operation that reads or writes outside the principal's default property scope requires an explicit cross-property grant, checked at the boundary. **[TP]**
2. A cross-property operation names its target explicitly in the contract, and records both source and target scope in its audit trail. **[TP]**

## 8. Schema and Migration Architecture

### 8.1 Sources of truth

1. Version control is the source of truth for schema and for all database-resident behaviour. The live database is a result, never a reference. **[EC — ARCH-010, ARCH-008]**
2. The repository declares it and the database enforces it — or the divergence is an explicit, reviewed property of the deployment, recorded where deployments are defined. Undeclared divergence is not a state the target allows. **[EC — Document 03 P2, P3]**

### 8.2 One chain per schema, one owner per chain

1. Each Prisma schema keeps its own migration chain; each chain has exactly one owning domain group and exactly one place where it is defined. **[EC — ARCH-009, ARCH-010: three schemas, a single migration owner]**
2. No migration is authored outside the chain, and a migration is never edited after it has been applied anywhere — corrections are new migrations. **[TP]**
3. Whether the three schemas remain three — and how the store survives for each duplicated concept — is governed by Document 03 P6 and is recorded here as **OD-07** at the granularity level (8.8). **[OD]**

### 8.3 Drift is a build failure

1. Three artefacts must agree at all times: the migration ledger recorded in version control, the migration directories on disk, and the state of the database under test. **[EC — ARCH-010, Document 03 P3]**
2. Each of the following fails the build rather than warning: a migration directory not recorded in the ledger; a ledger entry with no directory; a table referenced by models or code with no migration path to existence; a live table with no model and no documented legacy status; a ledger row claiming an applied migration that never ran. **[EC — ARCH-010: 28 of 51 migrations untracked, two unfinished ledger rows, one table dropped outside history, one table referenced 15 times and never created; ARCH-044: 14 live tables with no model]**
3. Tracked-but-deleted migration files, orphan SQL files, and generated artefacts committed to the repository are drift of the same kind and fail the same gate. **[EC — ARCH-045: 28 tracked-but-deleted files, orphan SQL migration files, tracked build artefacts, 40 stale compiled specs]**

### 8.4 Generated client reproducibility

1. The generated database client is a build output of committed schema: clean checkout plus generate equals the client in use. A client linked from a stale local directory is structurally impossible. **[EC — ARCH-046: a `file:`-linked inventory client declaring `VarChar(20)` against live and schema `uuid`, four models behind, imported at 20+ sites — Document 03 P5]**
2. Client regeneration is part of the normal build, and version skew between schema and client fails typecheck rather than surfacing as a runtime type mismatch. **[TP]**
3. Exactly one generated client per schema exists in the dependency graph; a second client over the same schema is forbidden (1.6). **[EC — ARCH-011, ARCH-017]**

### 8.5 Database-resident behaviour is migration-owned

1. Row-level policies, triggers, functions, extensions, and enum or domain types are schema. They are created, altered, and dropped only by migrations, and reviewed like code. **[EC — ARCH-008: 386 tables and 392 policies exist with no source; ARCH-041: triggers absent from every repository file; ARCH-020]**
2. Every policy declares its table, predicate, roles, and enforcement mode in migration source. "Exists in the live database only" is not an acceptable state for anything the application depends on. **[EC — ARCH-008]**
3. A trigger that mutates data is owned by the schema's owning context and either has an equivalent path through that context's contract or is declared legacy (14). **[EC — ARCH-041, ARCH-052]**

### 8.6 Evolution rules

1. One current release of the application runs against one current schema; changes are shaped so that running code and available schema do not require lockstep deployment. **[TP]**
2. Destructive change — drop, rename, type narrowing — is permitted only when no code path in the same release still references the old shape, enforced by typecheck and the drift gate (8.3), not by convention. **[TP]**
3. Where a change would break a running path, the schema change and the code change are each shaped to tolerate the other across the deploy window. **[TP]**

### 8.7 Environment parity

1. Every environment's schema is produced by replaying migrations from zero — never by copying a database. **[EC — ARCH-004: a disposable-schema Postgres harness with real migration replay — sound, preserve; ARCH-010]**
2. The same replay works for the test harness, local development, CI, and production. "Works on my database" is not a state. **[TP]**
3. The environment contract for database-gated suites — connection, database name, credentials — is defined in repository configuration and available to CI, so gated suites run where their gate applies instead of silently skipping. **[EC — Document 02 E-10: the availability test database URL is absent from CI; ARCH-024]**

### 8.8 Persistence granularity (open)

1. Schema-per-context — one schema per owning domain — is the required minimum granularity of isolation between domains' data. **[EC — ARCH-009]**
2. Whether any context, or the platform as a whole, escalates to a separate database — and which — is open as **OD-07**. **[OD]**

## 9. API Architecture

### 9.1 Three surfaces

1. The API has three surface types: **public** (clients), **internal** (in-process contract between contexts), and **integration** (external systems). They are distinct, and a route belonging to one is never reachable through another. **[TP]**
2. Internal contract surfaces MUST NOT be network-exposed; integration endpoints MUST NOT accept internal-only payloads; public endpoints MUST NOT expose internal contract types. **[TP]**
3. Public routes are versioned and consistently prefixed exactly once. A doubled or ambiguous prefix is a defect of the surface itself, not of its consumers. **[EC — Document 02 E-04, ARCH-048: 21 controllers resolve at `/api/v1/api/v1/...` and no consumer uses either form]**

### 9.2 Surface ownership

1. The context that owns the concept owns the endpoint that exposes it. No shared controller answers for another context's concept. **[EC — ARCH-013, ARCH-026]**
2. A public endpoint maps to exactly one application operation — one command or one query — in exactly one owning context. **[TP]**
3. Composition endpoints spanning two contexts are expressed as application-level orchestration that calls each context's contract in turn — never as a controller that writes two contexts' tables. **[TP]**

### 9.3 Command and query shape

1. One command = one handler = one transaction boundary (6.3). The existing command/query bus with its pipeline and duplicate-registration guard is the reference pattern and is preserved. **[EC — ARCH-003: sound]**
2. A query reads through its owning context's read path or an approved non-authoritative read model (6.5) — never another context's tables. **[TP]**
3. A handler returns a typed outcome. Reporting success when the operation did not occur — including when a downstream mechanism no-op'd — violates the contract. **[EC — ARCH-018]**

### 9.4 DTO and contract types

1. Each DTO is defined once, at the boundary that serves it, and consumers depend on that definition or its generated projection — never on a copy. **[EC — ARCH-026: duplicate DTOs for property and company; ARCH-033: two parallel reservation DTO sets both consumed]**
2. Contract types are not ORM entities: no field that exists only for persistence appears in a public DTO. **[TP]**
3. Server and client share domain vocabulary through the published contract. Three independent `Reservation` shapes across a client are forbidden. **[EC — ARCH-023]**
4. Request bodies are typed and validated at the boundary. An untyped body parameter is not a contract. **[EC — ARCH-033: 30 of 87 controllers accept `@Body() body: any`]**

### 9.5 Contract publication (open)

1. Exactly one machine-readable authoritative contract exists per public surface, and it is produced from the code that serves that surface. A contract that can drift from the implementation is not a contract. **[TP]**
2. Whether that contract is published as generated OpenAPI, as a shared typed package, or as both from one source is open as **OD-04**. **[OD]**
3. Whichever form is chosen, internal contract types and integration payload types are versioned with the same discipline, even though they are not public. **[TP]**

### 9.6 Authorization contract

1. Every mutating endpoint carries exactly one authorization requirement with unambiguous semantics, and an endpoint without one is not reachable: route construction makes an absent requirement impossible rather than relying on the author to remember it. **[EC — Document 03 P7, ARCH-016: both vocabularies are inert when their decorator is absent]**
2. There is exactly one permission vocabulary and one authorization mechanism evaluated per request. Two disjoint vocabularies both executing are forbidden (1.6). **[EC — ARCH-016: 47 and 173 sites, disjoint, both executing on every request]**
3. Which of the two existing mechanisms and vocabularies is retained — or whether a third is introduced — is open as **OD-09**. This document decides only that exactly one survives, is uniformly applied, is identity-aware, and fails closed. **[OD]**
4. Authorization is enforced at the surface boundary and, where it concerns scope, again at the database (7.4, 7.5). A check that exists only in the client does not exist. **[TP]**

### 9.7 Transport independence

1. A context's contract is defined independently of its transport; HTTP is a binding of the contract, not the contract itself. **[OD — see OD-06]**
2. Nothing in a contract's shape — naming, error model, pagination, idempotency — may depend on which process hosts it. **[TP]**

### 9.8 Errors and idempotency

1. Contract errors are typed, stable, and equivalent across surfaces: a failure the caller can act on is part of the contract, not an incidental string. **[TP]**
2. Every command whose retry could double-effect carries an idempotency key or equivalent de-duplication at the boundary, resolved within the owning context's transaction. Multiple parallel idempotency paths are forbidden (1.6). **[EC — ARCH-017: three idempotency paths]**
3. Failures inside a handler surface as errors, never as partial success. **[TP]**

## 10. Frontend Architecture

### 10.1 One contract surface for every client

1. Web, admin, and mobile are three clients of one contract surface. No client owns a private fork of the API, and no client-specific server behaviour exists. **[EC — Document 02 E-08: 278 web, 1 admin, 6 mobile call sites]**
2. A client's reach is a function of its identity and scope (7), not of separate endpoints. The same command from web and mobile executes identical server logic. **[TP]**
3. Whether a client ships, and with what surface, is a product and deployment decision — not an architecture fork. Admin and mobile are prototypes with opposite profiles, and their consumption must not drive contract divergence. **[EC — ARCH-036: admin deployed with one call site; mobile with six call sites and no image, manifest, or workflow]**

### 10.2 Server state and UI state are different things

1. Each application has exactly one server-state layer through which all remote reads and writes flow. **[EC — ARCH-021: four HTTP clients, three `ApiError` classes, five query-key registries with incompatible shapes for the same root key]**
2. UI state — filters, panels, drafts, selection — lives in UI stores and components; server state is never mirrored into it as a second source of truth. **[EC — ARCH-023: 13 of 20 global stores fetch server state]**
3. The same endpoint is fetched through one shared cache with one invalidation path. Double-caching the same data with no shared invalidation is forbidden. **[EC — ARCH-023]**
4. Domain types in a client come from the published contract (9.5). A client-local type with the same name and a different shape is a defect. **[EC — ARCH-023: three independent `Reservation` types]**

### 10.3 Contract-only access

1. Client code imports only published contract types, the single data layer, and UI code. Importing backend source, server internals, database clients, or another application's internals is forbidden (4.3.6). **[TP]**
2. Endpoints are addressed through one configuration of base URL and paths — no scattered hardcoded hosts and no fallbacks pointing at developer machines. **[EC — ARCH-035: nine hardcoded `localhost` fallbacks]**
3. A client never constructs a URL the contract does not declare, including doubled-prefix forms. **[EC — Document 02 E-04, ARCH-048]**

### 10.4 Feature boundaries are machine-enforced

1. Client features live in one uniform location per application, exposing an entry surface and not reaching into another feature's internals. **[EC — ARCH-022: three competing organisational schemes, a legacy-to-feature import cycle, 19 `app/` files reaching into feature internals]**
2. Linting covers all application code of every client, not a subdirectory of it, and boundary rules are lint rules — so a violation fails the build. **[EC — ARCH-031: lint covers only `app components lib`, leaving 612 of 871 web files unlinted]**
3. One application = one component and organisational convention. A second idiom introduced alongside the first is forbidden (1.6). **[EC — ARCH-022]**

### 10.5 Domain vocabulary in the UI

1. Concepts shared across features — reservation, guest, folio, availability — are named once per application and imported from the contract-backed definition. **[EC — ARCH-023]**
2. Components render business rules only through contract-shaped data and server-provided capabilities. A rule duplicated between client and server will diverge. **[TP]**

### 10.6 Secrets and session handling

1. No credential, signing secret, or long-lived key ships inside a client bundle; client configuration is public by definition. **[EC — ARCH-047: the JWT signing secret inlined into the client bundle with hardcoded fallbacks]**
2. Authentication state is handled in one place per application; duplicated auth logic across clients is forbidden (1.6). **[EC — ARCH-035: duplicated auth logic across web and admin]**
3. Cookie and session access goes through one wrapper, not copies. **[EC — ARCH-035: three `readCookie` copies]**

### 10.7 Scope of this document

1. This section constrains structure: state management, data access, feature boundaries, contract use. It makes no statement about screens, layouts, navigation, or interaction design; those are not part of the target architecture contract. **[TP]**

## 11. Integration Architecture

### 11.1 External systems live in adapters

1. Every external system — CRS, OTA and channel managers, payment gateways, notification providers — is reached only through an adapter owned by the domain that cares about it. **[TP]**
2. An external SDK or protocol client appears only inside its adapter; no business context imports external-system libraries (4.3.7). **[TP]**
3. Adapters translate between the external system's vocabulary and the owning domain's contract; neither vocabulary leaks into the other. **[TP]**

### 11.2 Inbound integrations

1. Inbound payloads are verified by signature or credential before any parsing or other work is done with them. **[EC — ARCH-006: HMAC verification and dedupe — sound, preserve]**
2. Duplicates are collapsed by a stable message identity; re-delivery is expected, not exceptional. **[EC — ARCH-006]**
3. After verification, the payload is expressed as a contract invocation into the owning context — create, update, or reject — and never as a direct write to that context's tables. **[EC — ARCH-006: creation through the reservations facade]**
4. A rejection — unverified, duplicate, unknown, unauthorised — is observable. It is not a silent drop. **[TP]**

### 11.3 Outbound integrations

1. Real and simulated outbound behaviour are distinguishable at runtime by configuration, by log and trace attributes, and by metric, so an operator can never mistake a simulation for a live transmission. **[EC — ARCH-038: payment gateway stub hardwired, push SDKs absent, channel metrics fabricated]**
2. A simulated path reports itself as simulated in every telemetry signal it emits. **[EC — ARCH-038]**
3. External rates, quotas, and errors surface as integration state — never fabricated into business metrics. **[EC — ARCH-038]**

### 11.4 External facts become internal facts

1. The owning context is the only one that converts an external message into an internal fact; other contexts learn of it through that published fact, never by receiving the external payload. **[TP]**
2. An external payload is never re-published verbatim as an internal fact (5.4). **[TP]**
3. Outbound translation happens in the adapter at send time; internal facts never carry another system's wire format. **[TP]**

### 11.5 Jobs and durable execution

1. Every background job belongs to exactly one owning domain: that domain defines what the job does, what its inputs mean, and what scope it runs in. **[TP]**
2. Exactly one job and queue execution mechanism exists. A second, parallel stack for the same kind of work is forbidden (1.6). **[EC — ARCH-032: three overlapping Temporal layers, one workflow never registered, an empty workflow barrel, queues nothing listens on; ARCH-018]**
3. A job type that is defined but never registered, or registered with no path that enqueues it, MUST NOT exist — the same class of defect as untracked migrations (8.3). **[EC — ARCH-018, ARCH-032, ARCH-042]**
4. Jobs receive explicit persisted scope (7.3), emit correlation identity, and surface failures to a retry or dead-letter state rather than dropping them (5.6). **[TP]**
5. Which mechanism carries jobs — and whether durable workflow execution is used at all — is part of **OD-03**. **[OD]**
6. Durable workflow execution, queue-based jobs, and in-process scheduling are three different capabilities. The target uses at most one mechanism per capability, and a mechanism is not reused across capabilities merely because it can perform them. **[TP]**

### 11.6 Payments and financial integrations

1. Payment gateways sit behind Billing. A gateway's state machine — authorise, capture, refund — never leaks upward as the domain's own vocabulary. **[TP]**
2. Billing owns folio and charge truth. Any other context requesting a charge does so through Billing's contract (6.1), per Document 03 P10. **[EC — ARCH-025, Document 03 P10]**
3. Financial callbacks and webhooks follow 11.2 exactly as OTA messages do: verify, dedupe, express as a contract invocation. **[TP]**

### 11.7 The no-bypass invariant

1. There is no path from an external system to an internal table, and no path from an internal context to an external system, that does not pass through an owning boundary — adapter or contract. **[TP]**
2. Integrations receive no elevated scope: an external caller's effective scope is established by verified credentials and the target's own identity rules, never by data in the payload. **[TP]**

## 12. Observability and Operational Architecture

### 12.1 One telemetry pipeline

1. Logs, traces, metrics, and request logging flow through one pipeline wired once, globally. A context does not construct its own logging stack, its own exporter, or its own sampler. **[EC — ARCH-005: OTel, pino, request logging, and exception filtering wired once, globally — sound, preserve]**
2. Exception handling reaches the surface through one filter path; an error is rendered once, not translated by several layers into several shapes. **[EC — ARCH-005, ARCH-017]**
3. A second telemetry mechanism introduced alongside the first is forbidden (1.6). **[TP]**

### 12.2 Correlation identity

1. One correlation identity is created per request or per job and is propagated across contract invocations, published facts, and job execution, so one business operation can be reconstructed across contexts. **[TP]**
2. Every log line, span, and audit record carries the identity, the owning context, and — where applicable and permitted — the scope under which it ran. **[TP]**
3. Facts and jobs carry correlation identity as part of their payload contract, not as an ambient global. **[TP]**

### 12.3 Structured signals

1. Operational signals are structured records, not ad-hoc console text; a signal that cannot be queried, filtered, or attributed is not an operational signal. **[TP]**
2. Business events are not inferred from log scraping: what must be auditable is recorded in the audit trail (12.6), what must be measurable is a metric. **[TP]**

### 12.4 Traces span context boundaries

1. A trace crosses context boundaries through contract invocations and fact consumption, so latency and failure in a cross-context flow are attributable to the context that caused them. **[TP]**
2. A trace never substitutes for the contract: an implementation must not be reconstructable only from traces. **[TP]**

### 12.5 Operational signals owned by the owning context

1. Each context publishes its own operational signals — throughput, failure rate, latency, and, where it has a queue, depth and dead-letter counts — and owns their meaning. **[TP]**
2. Integration adapters expose real-versus-simulated status as a first-class signal (11.3), so an operator can never mistake a simulation for a live transmission. **[EC — ARCH-038]**
3. Signals that are fabricated — metrics with no emitting system, counters with no source — are forbidden; a metric exists only where its underlying event exists. **[EC — ARCH-038: channel metrics fabricated]**

### 12.6 Auditability

1. There is exactly one audit mechanism for "who changed what, under which scope, when"; per-context ad-hoc audit writers are forbidden (1.6). **[EC — ARCH-017: two audit systems]**
2. Audit records are append-only (6.4) and are separate from operational logs: logs are for operating the system, audit is for answering for its changes. **[TP]**
3. Every state-changing contract invocation and every background mutation is auditable with actor, scope, target, and outcome. **[TP]**

### 12.7 Failure boundaries

1. A failure is attributable to exactly one owning context; the context that failed reports the failure, and no other context reports success on its behalf. **[EC — ARCH-018, ARCH-019]**
2. Silent success is forbidden at every layer: a path that did nothing reports that it did nothing (5.6, 9.3). **[TP]**
3. Errors crossing a contract boundary remain typed; telemetry wrapping must not erase the type the caller must handle. **[TP]**

### 12.8 Deployment posture is observable

1. The controls that are conditional or configurable — database isolation enforcement (7.5), simulated outbound paths (11.3), feature and worker toggles — are reported by the running system, not assumed from documentation. **[EC — ARCH-008, ARCH-038, ARCH-050: a disable flag absent from the deployment manifest left a worker enabled]**
2. An environment's isolation posture (which layer is active, for which role) is discoverable at runtime and recorded where deployments are defined. **[EC — Document 03 P2]**

## 13. Testing Architecture

### 13.1 Test strata

| Stratum | What it proves | Boundary rule |
|---|---|---|
| Domain / unit | Rules and state transitions in isolation | No database, no HTTP, no time or network dependence — **[TP]** |
| Integration | One context's behaviour against its real persistence, in a disposable schema | One context at a time; never another context's tables — **[TP]** |
| Contract | A published contract holds on both sides (provider and consumer) | Exercised through the contract, not through internals — **[TP]** |
| Boundary verification | Dependency direction, import rules, and feature boundaries hold mechanically | Fails the build on violation — **[EC — ARCH-031, 4.3]** |
| End-to-end | A business operation across contexts through public surfaces | Via public contract only; no fixture may write across contexts directly — **[TP]** |

1. A test never reaches into a context other than the one under test through non-contract means, in a unit or an assertion alike. **[TP]**

### 13.2 Isolation of test data

1. Database-backed tests run in disposable schemas created by replaying migrations from zero (8.7) and destroyed afterwards. They never point at a shared or long-lived database. **[EC — ARCH-004: transactional assertion engine plus a disposable-schema Postgres harness with real migration replay — sound, preserve]**
2. Test isolation is the same isolation as production: scope is present and enforced in test contexts too, so a test cannot pass by bypassing the boundary it claims to verify. **[TP]**

### 13.3 The environment contract

1. Every suite declares what it needs to run — service, database, environment variables — in repository configuration. **[EC — ARCH-024, Document 02 E-10]**
2. A gated suite whose gate is unavailable in the environment where it is supposed to run is a failure of that environment's configuration, not a pass. Silent skipping on the path that matters (the pull request) is forbidden. **[EC — Document 02 E-10: availability test database URL absent from CI; ARCH-024: DB-gated suites skip on every PR]**
3. The same suite definition runs in local development and in CI; two divergent test configurations are forbidden (1.6). **[TP]**

### 13.4 Tests are code

1. Test sources are included in the type gate: a test that imports a path that does not exist, or that types against a stale schema, fails the build. **[EC — ARCH-040: tests excluded from typecheck and transpiled with diagnostics off, one test already importing a nonexistent path; ARCH-024]**
2. Test fixtures are typed against the same generated client as the code under test, so a schema change breaks compiling tests as it breaks compiling code. **[EC — ARCH-046, ARCH-040]**

### 13.5 Boundaries are verified, not assumed

1. Dependency direction (4), cross-context import rules (5), contract-only client access (10.3), and feature boundaries (10.4) are enforced by machine checks that fail the build. **[EC — ARCH-031, ARCH-013]**
2. A rule in this document with no corresponding check is aspirational; each constraint in 15 states its enforcement locus where one exists today. **[TP]**

### 13.6 Capability honesty

1. A context's claimed capabilities are backed by executable evidence of those capabilities. Defined-but-unreachable handlers and stub modules are not capability (14.6). **[EC — ARCH-027: 19 of 37 modules stub or thin, 2 unregistered; ARCH-042]**
2. A behaviour change in a context carries a test that fails if that behaviour regresses. **[EC — Document 03 P11, ARCH-024: 26 of 37 modules untested]**

### 13.7 Evidence must be able to fail

1. A test that cannot fail, a suite that skips silently, and a gate that only warns do not constitute evidence and must not be counted as such. **[EC — ARCH-024, ARCH-039]**
2. Test results are reported where the gate runs; a green build that did not execute the suites it claims is a defect. **[EC — Document 02 E-10]**

### 13.8 Scope of this section (open)

1. This section fixes the structure of verification: strata, isolation, environment contract, type gate, boundary checks, and evidence. Coverage targets, framework selection, and suite organisation are not decided here and are recorded as **OD-10**. **[OD]**

## 14. Legacy Architecture Policy

### 14.1 Legacy is quarantined, not adopted

1. Superseded pipelines, abandoned tables, dormant writers, and unreachable modules exist in the target's inventory as **declared legacy** — they are not quietly treated as part of the current architecture. **[EC — ARCH-041, ARCH-043, ARCH-042, ARCH-045]**
2. Nothing in sections 2-13 grants legacy a path into a contract: legacy has no owning context among 2.2-2.5. **[TP]**

### 14.2 No new construction on legacy

1. New behaviour, new endpoints, and new consumers MUST NOT be built on a legacy store, pipeline, or code path. **[EC — ARCH-043: the legacy `inventory_*` pipeline still written by three modules; ARCH-041]**
2. A legacy path gains no second contract, no public surface, and no new importer. **[EC — ARCH-042]**

### 14.3 Legacy is identifiable

1. Every legacy table, pipeline, and module is declared as such where the repository records its inventory, so "legacy" is a status that can be queried rather than folklore. **[EC — ARCH-044: 14 live tables with no model; ARCH-045]**
2. The drift gate (8.3) accepts "documented legacy status" as the only alternative to a model — undocumented unmodelled tables fail the build. **[EC — ARCH-044, ARCH-010]**

### 14.4 Isolation requirements

1. Legacy code does not import current-context internals to appear alive, and current contexts do not import legacy internals; where a legacy path is still active, it sits behind the same adapter or contract rules as everything else (11). **[TP]**
2. Legacy data writers — triggers, jobs, scheduled statements — are enumerated, declared, and observable; an unenumerated database-resident writer is a defect (8.5). **[EC — ARCH-041, ARCH-052]**
3. No runtime fallback may silently switch from a current path to a legacy path; if a system can fall back, that fact is declared, observable, and bounded. **[EC — ARCH-038, ARCH-043]**

### 14.5 One authoritative pipeline where two exist

1. Where a legacy and a current pipeline write the same concept, exactly one is authoritative and the other is legacy by definition until reconciled; no contract, report, or rule may depend on the non-authoritative one. **[EC — ARCH-011, ARCH-043 — Document 03 P6]**
2. Neither pipeline may be extended to cover the other's gap while both remain (6.6). **[TP]**

### 14.6 Unreachable code is not architecture

1. Modules and handlers with no reachable path — unregistered, unimported, or routed to nothing — are not part of the target's structure and do not count toward capability or readiness. **[EC — ARCH-027, ARCH-042, ARCH-018, ARCH-032]**
2. The target contains no defined-but-unregistered handler, no registered-but-never-enqueued job, and no module that exists only in the module list (5.6, 11.5). **[EC — ARCH-018, ARCH-027, ARCH-032]**

### 14.7 Removal principle

1. Legacy is removed when nothing references it; while references exist it stays quarantined and observable rather than half-deleted. Tracked-but-deleted files, orphan migrations, and stale build artefacts are the opposite of this and are drift (8.3). **[EC — ARCH-045]**
2. The sequence, timing, and criteria for removal are not part of this document; this section defines the state legacy occupies in the target, not the route to it. **[TP]**

## 15. Architectural Constraints

Each constraint is a standing rule of the target state: **MUST** and **MUST NOT** apply to all work built on this architecture. Evidence tags show the basis: **[TP]** for target principles, **[EC]** for constraints fixed by current-state evidence, and **[OD]** for constraints whose final wording depends on an open decision in Appendix A.

### 15.1 Tenancy and isolation

| ID | Constraint | Basis |
|---|---|---|
| AC-01 | Scope for every non-public request MUST derive from authenticated identity; no client-supplied header, parameter, route value, or body field is authoritative for scope. | **[EC — ARCH-007; Document 03 P1]** |
| AC-02 | Exactly one isolation root and one scope parameter MUST be authoritative; setters, readers, guards, and policies MUST all express scope in that one root. | **[EC — ARCH-008; dependent on OD-01]** |
| AC-03 | Database-level isolation MUST be declared in this repository, apply to the exact role the application connects as, and be enforced by the storage engine — or its absence MUST be an explicit, reviewed property of that deployment. | **[EC — ARCH-008, ARCH-051; Document 03 P2; dependent on OD-02]** |
| AC-04 | Application authorization and database isolation MUST remain independent layers; misconfiguring one MUST NOT disable the other. | **[EC — ARCH-008, ARCH-016]** |
| AC-05 | Any operation outside a principal's property scope MUST carry an explicit cross-property grant checked at the boundary. | **[TP]** |

### 15.2 Boundaries and dependencies

| ID | Constraint | Basis |
|---|---|---|
| AC-06 | Cross-domain use of a context MUST occur only through the surface that context exports; no import may reach its internal layers. | **[EC — ARCH-013; Document 03 P4]** |
| AC-07 | Dependency direction MUST follow the layer model (4.1-4.3): no reverse edges into `modules`, no mutual synchronous dependence, no registration-time cycles. | **[EC — ARCH-029, ARCH-030]** |
| AC-08 | `platform`, `common`, and `core` MUST contain zero references to business-context state or modules. | **[EC — ARCH-001: sound, preserve]** |
| AC-09 | Exactly one mechanism MUST exist per cross-cutting concern (command bus, audit, policy engine, idempotency, error type, HTTP client, cache, scope context). | **[EC — ARCH-015, ARCH-016, ARCH-017, ARCH-021, ARCH-023]** |
| AC-10 | External-system SDK and protocol clients MUST appear only inside integration adapters of the owning domain. | **[TP]** |
| AC-11 | Client code MUST import only published contract types, its own data layer, and its own UI code. | **[TP]** |

### 15.3 Data and transactions

| ID | Constraint | Basis |
|---|---|---|
| AC-12 | Every business concept MUST have exactly one authoritative store and one owning context; no second writer to the same concept may exist. | **[EC — ARCH-011, ARCH-012, ARCH-025, ARCH-026; Document 03 P6, P9, P10]** |
| AC-13 | One application operation MUST equal one transaction; multi-step writes affecting money or inventory MUST complete within a single transaction boundary. | **[EC — ARCH-019, ARCH-025; Document 03 P8]** |
| AC-14 | A context MUST NOT read another context's tables, repositories, or database client by any path other than contract invocation or an approved read model. | **[EC — ARCH-014]** |
| AC-15 | Ledger and historical records MUST be append-only; correction MUST occur by new entry referencing the original. | **[EC — ARCH-044, ARCH-019]** |
| AC-16 | Derived read models MUST be explicitly non-authoritative, single-producer, and never the basis of a business rule's trust. | **[TP]** |
| AC-17 | Cross-context atomicity MUST NOT use distributed transactions; facts and compensations are the mechanism. | **[TP]** |

### 15.4 Schema and migrations

| ID | Constraint | Basis |
|---|---|---|
| AC-18 | The migration ledger, the migration directories, and the database under test MUST agree; any disagreement MUST fail the build. | **[EC — ARCH-010; Document 03 P3]** |
| AC-19 | All schema change MUST occur through migrations; application code MUST NOT create or alter schema at boot or at runtime. | **[EC — ARCH-020]** |
| AC-20 | All database-resident behaviour — policies, triggers, functions — MUST be declared in migrations and owned by the schema's owning context. | **[EC — ARCH-008, ARCH-041]** |
| AC-21 | The generated database client MUST be reproducible from committed schema; schema-to-client skew MUST fail typecheck; exactly one client per schema MUST exist. | **[EC — ARCH-046, ARCH-011; Document 03 P5]** |
| AC-22 | Every environment's schema MUST be produced by replaying migrations from zero. | **[EC — ARCH-004; Document 02 E-10]** |

### 15.5 API

| ID | Constraint | Basis |
|---|---|---|
| AC-23 | Every mutating endpoint MUST carry exactly one authorization requirement with unambiguous semantics; an endpoint without one MUST NOT be reachable. | **[EC — ARCH-016; Document 03 P7]** |
| AC-24 | Exactly one permission vocabulary and one authorization mechanism MUST be evaluated per request. | **[EC — ARCH-016; dependent on OD-09]** |
| AC-25 | Exactly one machine-readable contract MUST exist per surface, produced from the code that serves it. | **[TP; dependent on OD-04]** |
| AC-26 | Contract types MUST be defined once at the owning boundary; duplicated DTO sets and untyped request bodies MUST NOT exist. | **[EC — ARCH-026, ARCH-033]** |
| AC-27 | Public routes MUST be versioned and prefixed exactly once; a doubled or ambiguous prefix MUST NOT exist. | **[EC — ARCH-048; Document 02 E-04]** |
| AC-28 | Retryable commands MUST carry idempotency handling at the boundary, and exactly one idempotency mechanism MUST exist. | **[EC — ARCH-017]** |
| AC-29 | Internal contract surfaces MUST NOT be network-exposed; integration endpoints MUST NOT accept internal-only payloads. | **[TP]** |

### 15.6 Frontend

| ID | Constraint | Basis |
|---|---|---|
| AC-30 | Each application MUST have exactly one server-state layer, kept separate from UI state, with one cache and invalidation path per endpoint. | **[EC — ARCH-021, ARCH-023]** |
| AC-31 | Domain types in clients MUST come from the published contract; parallel client-local shapes MUST NOT exist. | **[EC — ARCH-023]** |
| AC-32 | Feature boundaries MUST be machine-enforced, and lint MUST cover all application code of every client. | **[EC — ARCH-022, ARCH-031]** |
| AC-33 | No credential, signing secret, or long-lived key MUST exist in a client bundle, and each application MUST handle auth state in exactly one place. | **[EC — ARCH-047, ARCH-035]** |

### 15.7 Integration and jobs

| ID | Constraint | Basis |
|---|---|---|
| AC-34 | Inbound external payloads MUST be verified before parsing, deduplicated by stable identity, and expressed as contract invocations — never as direct writes. | **[EC — ARCH-006: sound, preserve]** |
| AC-35 | Simulated outbound behaviour MUST be distinguishable from real behaviour in every telemetry signal it emits. | **[EC — ARCH-038]** |
| AC-36 | Exactly one job and queue execution mechanism MUST exist; every job MUST belong to exactly one owning domain; defined-but-unregistered and registered-but-unreachable jobs MUST NOT exist. | **[EC — ARCH-018, ARCH-032, ARCH-042]** |
| AC-37 | No path MUST exist from an external system to an internal table, or from an internal context to an external system, that bypasses an owning adapter or contract. | **[TP]** |
| AC-38 | Fact publication MUST occur inside the owning context's transaction boundary, with exactly one publication path that demonstrably carries facts. | **[EC — ARCH-018, ARCH-019]** |

### 15.8 Observability, verification, and legacy

| ID | Constraint | Basis |
|---|---|---|
| AC-39 | One telemetry pipeline and one audit mechanism MUST exist; both MUST carry correlation identity across contract invocations, facts, and jobs. | **[EC — ARCH-005: sound, preserve; ARCH-017]** |
| AC-40 | Database-gated suites MUST run where their gate applies; silent skipping MUST NOT constitute a pass, and test sources MUST be included in the type gate. | **[EC — ARCH-024, ARCH-040; Document 02 E-10]** |
| AC-41 | A behaviour change MUST carry executable evidence that fails if the behaviour regresses. | **[EC — Document 03 P11, ARCH-024]** |
| AC-42 | Legacy stores, pipelines, and paths MUST NOT receive new behaviour, endpoints, or consumers; where two pipelines exist, exactly one MUST be authoritative. | **[EC — ARCH-041, ARCH-042, ARCH-043; Document 03 P6]** |
| AC-43 | Legacy status MUST be declared where the repository records its inventory; undocumented unmodelled tables MUST fail the drift gate. | **[EC — ARCH-044, ARCH-045]** |

### 15.9 Mapping to Document 03 preconditions

| Precondition | Satisfied by | Note |
|---|---|---|
| P1 — scope from authenticated identity | AC-01, AC-04, AC-05 | Global precondition; AC-01 is its literal wording |
| P2 — isolation policy declared in-repo for the connecting role | AC-03, AC-04, AC-20, AC-39 | Global precondition; mechanism choice is OD-02 |
| P3 — schema and migration baseline recoverable and consistent | AC-18, AC-19, AC-21, AC-22, AC-43 | Global precondition |
| P4 — cross-domain use of reservations through its exported surface | AC-06, AC-07 | Global precondition; holds for every context, not only reservations |
| P5 — inventory client reflects current schema and live columns | AC-21 | Domain-scoped (inventory) |
| P6 — single authoritative store per purchasing/stock/warehouse concept | AC-12, AC-42 | Domain-scoped (inventory, purchasing) |
| P7 — every mutating endpoint carries an unambiguous authorization requirement | AC-23, AC-24 | Applies to any new API surface |
| P8 — multi-step money or inventory writes in one transaction | AC-13, AC-15 | Money- and inventory-affecting paths |
| P9 — single authoritative guest identity record | AC-12 | Guest-identity-facing work |
| P10 — folio and charge data has a single writer | AC-12, AC-13 | Folio/billing work |
| P11 — behaviour change carries executable evidence | AC-40, AC-41 | The 26 modules with no tests |

### 15.10 How these constraints are fixed

1. Constraints tagged **[TP]** hold independently of any open decision. **[TP]**
2. Constraints tagged **[EC]** are fixed by current-state evidence and do not loosen if an open decision resolves differently. **[TP]**
3. Constraints whose wording depends on an open decision — AC-02 (OD-01), AC-03 (OD-02), AC-24 (OD-09), AC-25 (OD-04) — hold their invariant now and are re-expressed once the decision closes; the invariant itself is not contingent. **[OD]**

## 16. Target-State Definition

### 16.1 What is true in the target

| Dimension | The target state | Constraints |
|---|---|---|
| Structure | Contexts with single ownership of concepts, state, and contracts; dependencies point one way by strata; boundaries enforced by machine, not convention | AC-06 to AC-11 |
| Data | One authoritative store per concept; one transaction per operation; append-only history; derived data clearly non-authoritative | AC-12 to AC-17 |
| Tenancy | Scope derived from identity, one isolation root, two independent enforcement layers, database isolation declared in-repo and enforced for the connecting role | AC-01 to AC-05 |
| Schema | One migration chain per schema with one owner; ledger, disk, and database agree or the build fails; no runtime DDL; reproducible generated clients | AC-18 to AC-22 |
| API | One contract per surface produced from serving code; one authorization mechanism; typed DTOs, errors, and idempotency; internal surface not network-reachable | AC-23 to AC-29 |
| Frontend | One contract surface for all clients; one server-state layer per app; feature boundaries lint-enforced; no secrets in bundles | AC-11, AC-30 to AC-33 |
| Integration | Adapters own external systems; inbound verified and deduped; simulated distinguishable from real; one job mechanism with per-domain job ownership | AC-34 to AC-38 |
| Operations | One telemetry pipeline, one audit mechanism, correlation across boundaries, deployment posture observable at runtime | AC-39 |
| Verification | Test strata with disposable-schema isolation, an honoured environment contract, tests inside the type gate, boundaries checked mechanically | AC-40, AC-41 |
| Legacy | Declared, quarantined, excluded from new construction, never a silent fallback, one authoritative pipeline where two exist | AC-42, AC-43 |

### 16.2 The definition in one statement

The target is a multi-context system in which each business concept has exactly one owner, one authoritative store, and one exported surface; in which every request's scope comes from its identity and is enforced twice; in which schema is reproducible from version control alone; in which clients, contexts, and external systems interact only through contracts and published facts; and in which every boundary in this document is checked by the build rather than trusted to convention — with legacy present only as declared residue that receives nothing new.

### 16.3 What this document does not settle

1. Ten decisions remain open (OD-01 to OD-10, Appendix A). Each is recorded with its statement, the evidence that keeps it open, what would close it, and the sections it touches. **[OD]**
2. Relations held open deliberately: P1 and P4 appear here only as the invariants they require (15.9); the reservations domain's internal design is out of scope by 3.4. **[TP]**
3. Nothing in this document assigns work, sequences changes, or evaluates who performs them; that is a different class of document. **[TP]**

## Appendix A — Open Decisions

| ID | Decision | Primary sections | Depends on |
|---|---|---|---|
| OD-01 | The tenancy hierarchy and isolation root | 7.1, 7.7, 7.8, 15.1 | Identity model, tenant/property master data |
| OD-02 | The database-level isolation mechanism | 7.4, 7.5, 15.1 | OD-01, deployment topology (OD-06) |
| OD-03 | The cross-domain communication and job-execution mechanism | 4.4, 5.7, 11.5, 12.1 | OD-06 |
| OD-04 | The contract publication mechanism | 9.5, 10.3, 15.5 | OD-06 |
| OD-05 | The reporting and analytics data path | 2.6, 6.5 | OD-07 |
| OD-06 | Deployment and process topology | 5.2, 5.7, 9.7 | Capacity and operational requirements |
| OD-07 | Persistence granularity — schema-per-context versus database-per-context | 6.2, 8.2, 8.8 | OD-01 |
| OD-08 | The cross-domain identifier convention | 6.7, 9.4 | OD-07 |
| OD-09 | Which authorization mechanism and permission vocabulary survive | 9.6, 15.5 | Document 03 P7 wording |
| OD-10 | Test strategy specifics — strata weighting, framework selection, suite organisation | 13.8 | Document 03 P11 wording |

### A.1 OD-01 — Tenancy hierarchy and isolation root

- **Statement.** Is there a tenant level above properties, or is the property itself the isolation root — and how do properties, hotels, and corporate or chain accounts nest beneath whatever root is chosen?
- **Why it is open.** The repository expresses scope two ways at once: setters write `app.current_tenant` while policies read `app.hotel_id`, and both `tenant_id` and `hotel_id` naming appear across models. Evidence establishes that exactly one root is required (AC-02) but cannot say which concept is that root. **[EC — ARCH-008]**
- **What closes it.** A decision grounded in the tenant and property master data — who buys, who stays, who is billed, which entity the contract signs — recorded as a single naming and keying rule for scope everywhere.
- **Sections affected.** 7.1, 7.7, 7.8, 8.8, 15.1 (AC-01, AC-02), 15.3 (AC-12 keying).

### A.2 OD-02 — Database-level isolation mechanism

- **Statement.** Which mechanism enforces isolation at the storage layer: row-level security with enforced policies, application-enforced query scoping with a compensating control, physical separation of tenant data, or a combination — and for which role and connection path.
- **Why it is open.** 386 tables and 392 policies exist live with zero forced enforcement, under a role that is owner, superuser, and bypass-capable, with setters and readers disagreeing on the scope parameter; the repository holds one policy file and it is absent live. The evidence proves the current arrangement provides no isolation; it does not select the replacement. **[EC — ARCH-008, ARCH-051]**
- **What closes it.** A choice among mechanisms with its enforcement mode, connection role, and declaration location fixed — satisfying AC-03 — plus a decision on whether any deployment may consciously run with the second layer absent.
- **Sections affected.** 7.4, 7.5, 15.1 (AC-03, AC-04), 12.8, 8.7.

### A.3 OD-03 — Cross-domain communication and job-execution mechanism

- **Statement.** What carries published facts and background jobs: an in-process bus, a message broker, durable workflow execution, or a defined division of labour among them — and whether durable execution is used at all.
- **Why it is open.** The repository contains at least five candidate carriers and none is demonstrably the one: an in-process dispatcher with zero registrations, an outbox processor with no callers, an unregistered analytics consumer, three overlapping Temporal layers whose single workflow was never registered, queues nothing listens on — and only the BullMQ `events` leg carrying traffic. **[EC — ARCH-018, ARCH-032; Document 02 E-09]**
- **What closes it.** A selection that assigns each capability — inter-context facts, job execution, durable multi-step orchestration — exactly one mechanism, with the registration and reachability rules of 11.5 attached to it.
- **Sections affected.** 4.4, 5.5, 5.7, 11.5, 12.1, 15.7 (AC-36, AC-38).

### A.4 OD-04 — Contract publication mechanism

- **Statement.** How the authoritative contract is published to consumers: generated OpenAPI from serving code, a shared typed package, or both derived from a single source.
- **Why it is open.** The requirement for one non-drifting contract is fixed (AC-25), but the repository has no single publication path today — and surfaces carry consumers that reach no route (doubled prefixes), so the shape of publication cannot be inferred from usage. **[EC — Document 02 E-04, E-07; ARCH-048]**
- **What closes it.** A choice of publication artefact and the build step that produces it from the serving code, covering public, internal, and integration surfaces alike.
- **Sections affected.** 9.5, 10.3, 12.2, 15.5 (AC-25), 15.2 (AC-11).

### A.5 OD-05 — Reporting and analytics data path

- **Statement.** Whether reporting and analytics consume cross-context read contracts or rebuildable projections — and which store the projections are built into.
- **Why it is open.** Supporting capabilities currently hold no authoritative business state (2.6), but the repository gives no evidence of an existing reporting path to extend or replace; the choice determines whether reporting reads contracts (looser coupling, more calls) or projections (extra pipeline, no cross-context reads). **[OD]**
- **What closes it.** A decision naming the read path for each reporting class (operational reads, historical analysis, exports), with the rule that no report becomes the trust basis for a business rule (6.5).
- **Sections affected.** 2.6, 6.5, 6.2, 15.3 (AC-16).

### A.6 OD-06 — Deployment and process topology

- **Statement.** Are contexts hosted in one process, per-stratum processes, or per-context services — and does any contract binding become network-transported?
- **Why it is open.** The repository is monolithic in composition with a single physical database, but nothing in the evidence fixes where the target draws process boundaries; the answer changes the transport of 5.2 and 9.7, the reachability of the internal surface (AC-29), and the constraints that apply to OD-02 and OD-03. **[EC — ARCH-009: one physical database; ARCH-001: single composition root]**
- **What closes it.** A topology decision stating process boundaries and what crosses them, with the contract-then-transport rule of 9.7 attached.
- **Sections affected.** 5.2, 5.7, 9.7, 11.5, 12.1, 15.5 (AC-29).

### A.7 OD-07 — Persistence granularity

- **Statement.** Is schema-per-context the final granularity of data separation, or does any context escalate to its own database?
- **Why it is open.** Three Prisma schemas sit inside one physical database under a single migration owner, with cross-schema keys as untyped strings; schema-per-context is the required minimum (AC-18 chain ownership), but the evidence neither requires nor rules out database-level separation. **[EC — ARCH-009]**
- **What closes it.** A per-context granularity decision, taken together with OD-01 and OD-06, that fixes how many chains exist and what a cross-context reference physically is (OD-08).
- **Sections affected.** 6.2, 6.7, 8.2, 8.8, 15.4.

### A.8 OD-08 — Cross-domain identifier convention

- **Statement.** What a cross-domain reference looks like: format, naming, mutability, and whether identifiers are globally unique or scoped by isolation root.
- **Why it is open.** The repository mixes UUID primary keys, text keys, and composite keys, and cross-schema keys are untyped strings; typecheck alone cannot choose a convention, and the choice interacts with the isolation root (OD-01) because a reference's meaning must be scoped. **[EC — ARCH-009]**
- **What closes it.** A single rule for how one context names another context's records in contracts, foreign keys, and audit trails.
- **Sections affected.** 6.7, 7.7, 9.4, 15.3 (AC-12 keying).

### A.9 OD-09 — Authorization mechanism and permission vocabulary

- **Statement.** Which of the two existing authorization mechanisms and permission vocabularies is retained — or whether a third, unified vocabulary is introduced — and how permissions are named.
- **Why it is open.** Two disjoint vocabularies (47 and 173 sites) both execute on every request and both go inert when their decorator is absent; the sequence itself is sound and fail-closed, but nothing in the evidence decides which vocabulary describes the system's real permissions. Document 03 explicitly left this undecided. **[EC — ARCH-016; Document 03 P7 and non-decision 5]**
- **What closes it.** A choice of one mechanism and one vocabulary, with the enforcement-locus rule of AC-23 attached so an endpoint cannot exist without a requirement.
- **Sections affected.** 7.2, 9.6, 15.1, 15.5 (AC-23, AC-24).

### A.10 OD-10 — Test strategy specifics

- **Statement.** Coverage targets, framework or runner selection, and suite organisation across the four applications and the API.
- **Why it is open.** The structure of verification is fixed by 13.1-13.7 (strata, disposable schemas, environment contract, type gate, boundary checks, evidence), but the repository holds no resolved strategy: DB-gated suites skip on every pull request, 26 of 37 modules carry no tests, tests are excluded from typecheck, and the documented end-to-end command points at files that do not exist. Document 03 explicitly left strategy undecided. **[EC — ARCH-024, ARCH-040, ARCH-049; Document 02 E-10; Document 03 non-decision 10]**
- **What closes it.** A strategy decision naming the strata each change class must satisfy, where suites run, and how gates are reported — attaching AC-40 and AC-41 to those gates.
- **Sections affected.** 8.7, 13.1 to 13.8, 15.8 (AC-40, AC-41).

---

**End of Document 04 — Target Architecture.** Next document in the sequence: the gap analysis, which compares this target against the current state recorded in Documents 01-03. This document itself assigns no work and sequences nothing.

XYLO HOUSEKEEPING — PHASE 1 FORENSIC AUDIT CONTROL RECORD
Field	Value
Document	00_AUDIT_CONTROL.md
Lifecycle step	Phase 1 — Forensic Audit (before Business Rules Decision)
Status	COMPLETE (see 06_PHASE1_EXIT_ASSESSMENT.md for the gate decision)
Date	2026-10-20
Author	Forensic audit session (AI agent, read-only investigation)
Code changes	None. Zero production, test, schema, migration, config, or dependency modifications. Only this document set under docs/audit/Housekeeping/ was created.
1. Repository state at audit time
Field	Value
Repo root	C:\Users\Pro\Desktop\XYLO
Branch	main
HEAD commit	ed278bc3a6817351c5b358292109828c412f365c — "fix(db): C-03 - recover reproducible migration baseline (GAP-06/ARCH-010/044/045)" (2026-10-09 00:19:52 +0300)
History depth	3 commits total (first commit → Update application files → c854f79 Phase 0 - Enterprise Platform Foundation complete → ed278bc…). The Housekeeping module was never individually committed after Phase 0.
Working tree	Substantially uncommitted. Repo-wide: 414 modified, 243 deleted, 4 added, 228 untracked paths. Audit scope paths alone: 41 modified, 2 deleted (modules/housekeeping/dto.ts, modules/housekeeping/roles.guard.ts), 2 added, 15 untracked (the CQRS person-discrepancy directories and features/housekeeping/).
Audit basis	The on-disk working tree as found. All file/line references are to working-tree content, not to any committed revision.
Preservation	No file was modified, staged, reset, stashed, or cleaned. The working tree was only read (git status, git log, file reads, greps, line counts).
Housekeeping-scoped working-tree status (evidence: git status --porcelain -- apps/api/src/modules/housekeeping apps/web/store/housekeepingStore.ts "apps/web/app/(dashboard)/housekeeping" apps/web/features/housekeeping …):

 D apps/api/src/modules/housekeeping/dto.ts
 D apps/api/src/modules/housekeeping/roles.guard.ts
 M apps/api/src/modules/housekeeping/housekeeping.controller.ts
 M apps/api/src/modules/housekeeping/housekeeping.module.ts
 M apps/web/app/(dashboard)/housekeeping/_components/*.tsx (all 12)
 M apps/web/app/(dashboard)/housekeeping/page.tsx
 ?? apps/api/src/modules/housekeeping/application/
 ?? apps/api/src/modules/housekeeping/domain/
 ?? apps/api/src/modules/housekeeping/events/
 ?? apps/api/src/modules/housekeeping/infrastructure/
 ?? apps/web/features/housekeeping/
 ?? apps/web/features/front-office/components/dashboard/HousekeepingSummary.tsx
The two deleted backend files (dto.ts, roles.guard.ts) match verdicts in docs/reservations/phase-3/01_R8_CLASSIFICATION_LEDGER.md:180-184 (rows 52-56, DELETE), i.e. the working tree reflects an already-approved dead-code sweep.

2. Audit boundaries (established, not assumed)
In-scope discovery areas:

apps/api/src/modules/housekeeping/ — controller (62 routes), 4 services, CQRS command stack, events.
apps/web/app/(dashboard)/housekeeping/ — route, page, 12 _components files (6 live, 6 dead).
apps/web/store/housekeepingStore.ts — legacy Zustand store (1,523 lines).
apps/web/features/housekeeping/ — React Query person-discrepancy feature.
Persistence: packages/db/schema.prisma housekeeping models; packages/db/migrations/20260612000000_add_housekeeping_tables, 20260705040000_add_housekeeping_supply_requests, baseline/reconciliation migrations; packages/db/migration-baseline.declarations.json.
Cross-module seams actually touched by housekeeping code: front-office (due-outs, discrepancy comparator), room-service (guest requests), lost-found (items/settings/storage/claims), employees/HR reference data, command-center widgets.
Authorization surfaces: PropertyScope, PropertyScopeGuard/Resolver, TenantInterceptor, JwtAuthGuard, permission registry roles.
Out of scope (per Phase 1 constraints): Front Office functional audit, Room Service audit, Lost & Found audit, Inventory supply-request audit (except the /housekeeping/supply-requests surface), UI redesign, business-rule decisions, target-domain specification.

3. Method
Static reading and tracing of source; repo-wide greps for routes, endpoints, status literals, table names.
Cross-referencing of three independent evidence classes (source inspection, prior executed audit phases in docs/reservations/ + packages/db/migration-baseline.declarations.json, and render/import graphs).
Six parallel investigation tracks (frontend, API contract, room-status semantics, checkout integration, tests, permissions/tenancy) with findings reconciled against each other and spot-verified by the lead agent.
No runtime execution was performed against any system (see §4).
Evidence labels used in this document set (house style, mirroring docs/enterprise/availability-phase3-reservation-lifecycle-forensic-audit.md):

VERIFIED FACT — directly confirmed in repository code, schema, migrations, or ratified declarations.
OBSERVED BEHAVIOR — code path as written (source-inspected only; not proven at runtime).
INFERENCE — reasonable conclusion requiring confirmation.
UNVERIFIED / OPEN — insufficient evidence.
4. Runtime verification status and limitations
Item	Status
API runtime	Not started by this audit. No dev server, docker compose, DB connection, or authenticated request was executed.
Database	Not connected. Table presence claims rest on migration chain + migration-baseline.declarations.json (ratified 2026-10-09, commit same day as HEAD + c854f79 chain).
Web UI	Not executed. Component behavior claims are render-graph + source based.
Existing tests	Inspected only. No test was executed (execution decision deferred to 05_VERIFICATION_AND_COVERAGE.md conclusions).
This limitation is acceptable for Phase 1's purpose (establish verified current state from repository evidence) but is explicitly recorded: every "OBSERVED BEHAVIOR" in this document set is source-inspection evidence, not runtime-verified evidence. Where a later gate (readiness review, implementation) needs runtime proof, the verification gap must be closed there.

5. Deliverables index
File	Deliverable
00_AUDIT_CONTROL.md	This control record
01_EXECUTIVE_AUDIT_REPORT.md	Deliverable 1 — Executive Audit Report
02_FINDINGS_REGISTER.md	Deliverable 2 — Evidence-Based Findings Register
03_WORKFLOW_MAPS.md	Deliverable 3 — Current-State Workflow Maps
04_DOMAIN_AND_DATA_MAP.md	Deliverable 4 — Current-State Domain and Data Map
05_VERIFICATION_AND_COVERAGE.md	Deliverable 5 — Verification and Coverage Report
06_PHASE1_EXIT_ASSESSMENT.md	Deliverable 6 — Phase 1 Exit Assessment
Finding IDs: HK-### sequential. Categories per the audit mandate: CONFIRMED DEFECT (D) / DESIGN-ARCHITECTURE RISK (R) / UX-FUNCTIONAL GAP (G) / INCONSISTENCY (I) / ENHANCEMENT CANDIDATE (E) / UNVERIFIED-OPEN QUESTION (Q).

6. Prior audit lineage already on record for this module
The following previously executed audit phases constrain or corroborate this audit (referenced, not re-audited):

docs/reservations/phase-3/01_R8_CLASSIFICATION_LEDGER.md — verified live vs dead endpoints; housekeeping dto.ts, roles.guard.ts, 3 value-objects deleted; features/housekeeping turndown-attributes/credit-rules/membership-scheduling tasks deleted (B5-B7); GET/PUT /housekeeping/config/sla + status-flow verified live (108).
docs/reservations/phase-1/02_PHASE1_CLOSURE_AND_ROADMAP.md:84 — B4.3 deletion ledger execution; X6 room-configs store actions retained because live UI callers exist.
packages/db/migration-baseline.declarations.json:89-99 — person_discrepancies formally declared code-referenced-never-created, acceptedKnownDefect: true.
docs/enterprise/opera-cloud-gap-analysis-2026-08-29.md:22 — OPERA parity expectation list for Housekeeping (documentary only; treated as context, not authority).

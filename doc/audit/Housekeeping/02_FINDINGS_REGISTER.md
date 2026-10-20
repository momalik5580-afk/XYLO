# XYLO HOUSEKEEPING — EVIDENCE-BASED FINDINGS REGISTER (DELIVERABLE 2)

Document: 02_FINDINGS_REGISTER.md
Lifecycle step: Phase 1 — Forensic Audit
Status: COMPLETE
Date: 2026-10-20
Evidence basis: working tree at commit `ed278bc3` + uncommitted state (see 00_AUDIT_CONTROL.md §1).

## Reading guide

- Categories: **[D] CONFIRMED DEFECT** (violates an existing enforced contract / verified requirement — NOT merely a preferred design) · **[R] DESIGN / ARCHITECTURE RISK** (evidenced structural condition) · **[G] UX / FUNCTIONAL GAP** (demonstrable interface-vs-verified-requirement mismatch) · **[I] INCONSISTENCY** (conflicting definitions/behavior across layers) · **[E] ENHANCEMENT CANDIDATE** (not yet an approved requirement) · **[Q] UNVERIFIED / OPEN QUESTION**.
- Severity: HIGH / MEDIUM / LOW where evidence supports it; consequences stated operationally.
- Confidence: HIGH (multiple independent evidence sources or direct read) / MEDIUM (single-source code reading).
- Paths are repo-relative; lines are working-tree line numbers.
- All claims are source-inspection evidence; runtime status in 05_VERIFICATION_AND_COVERAGE.md.

---

## A. Confirmed defects

### HK-001 [D] — Large share of the Housekeeping frontend calls endpoints that do not exist server-side (severely degraded, persistence-free workflows)

- **Evidence (both sides read in full):** backend has exactly 62 routes in `apps/api/src/modules/housekeeping/housekeeping.controller.ts` (:19–427) + nothing under other prefixes; frontend calls enumerated in `apps/web/store/housekeepingStore.ts`.
- **Missing endpoints actually invoked by LIVE UI** (route → caller):
  - `POST /housekeeping/room-assignments/move-room` ← `housekeepingStore.ts:814` via `DailyAssignmentsSection.tsx:702–769,344–349` (move-room popover is rendered per room) — service method exists un-routed (`housekeeping.service.ts:262–284`)
  - `POST/PUT/DELETE /housekeeping/linen/types[/:id]` ← store `1263/1274/1286` (only GET exists, controller `:76`)
  - `PUT /housekeeping/linen/locations/:id` ← store `1313`
  - `GET/POST /housekeeping/linen/movements` ← store `1327/1344`
  - `GET/POST/PUT/DELETE /housekeeping/linen/room-configs` ← store `1355/1365/1376/1388`
  - `GET/POST/PUT /housekeeping/linen/damaged`, `/housekeeping/linen/lost` ← store `1398–1458`
  - `POST /housekeeping/linen/transactions`, `PUT /housekeeping/linen/stock/:id` ← store `938/948`
  - `POST /housekeeping/laundry`, `PUT /housekeeping/laundry/:id/status`, `DELETE /housekeeping/laundry/:id`, `POST /housekeeping/laundry/:id/inspect` ← store `985/962/995/1005` (no laundry routes at all)
  - `POST /housekeeping/supplies/transactions`, `PUT /housekeeping/supplies/stock/:id` ← store `1080/1090` (unused by live UI)
  - `POST /housekeeping/requests`, `PUT /housekeeping/requests/:id/acknowledge|complete` ← store `869/840/855` (unused by live UI)
  - `GET /housekeeping/teams` ← store `1134` (unused)
  - `POST /lost-found/items/:id/claim` ← store `1050` (backend flow is `POST /lost-found/claims`, `lost-found.controller.ts:152`)
- **Operational consequence:** every Linen & Laundry write, room movements, damaged/lost linen, linen types/locations edits, room moves between attendants, and the (latent) front-desk request actions touch dead URLs → HTTP 404. Because `LinenSection.tsx` never renders the store `error` and the move-room modal never renders it either (frontend track §10), all these failures are **silent**; data remains client-session-only.
- **Verification method:** full route sweep of `@Controller(` across `apps/api/src` (90 controllers) crossed with every URL string in `apps/web` (per contract track Part 3 table).
- **Confidence:** HIGH.

### HK-002 [D] — The refill/daily linen issue loop into movements is fabricated client-side and never persisted

- **Evidence:** `housekeepingStore.ts:635–744` — on `updateTaskStatus('in-progress'|'completed')` the store builds `LinenMovement` records (`id: LMV-${Date.now()}-replenish-${i}`) from a hardcoded per-room-type `DEFAULTS` table (`637–695`) and writes them **only into local state** (`set({ linenMovements: … })`). No API exists to persist (see HK-001). `LinenSection.tsx` renders these fabricated movements.
- **Consequence:** linen replenishment/collection on room cleaning is cosmetic: numbers appear, nothing is counted against `linen_inventory`.
- **Confidence:** HIGH.

### HK-003 [D] — Person-discrepancy feature (the only CQRS-built feature) is broken end-to-end: 404 read, payload mismatches, missing table, zero UI consumer

- **Evidence:**
  1. `GET /housekeeping/discrepancies` is called (`features/housekeeping/api/person-discrepancy.api.ts:29–31`) but **no such route exists** (controller has only `mobile/discrepancies`).
  2. `PUT /housekeeping/rooms/:roomNumber/person-count` expects `hkPersonCount` (`housekeeping.controller.ts:342,376`); the frontend sends `{ personCount }` (`person-discrepancy.api.ts:39–42`) → handler receives `undefined` (`record-hk-person-count.handler.ts:26–39`).
  3. Resolve endpoints expect `{ action, reason }` (`controller:392,411`) — `action` is cast `as any` with **no runtime validation**; the frontend sends only `{ resolutionNote }` (`person-discrepancy.api.ts:45–62`) → `action` undefined.
  4. `GET /housekeeping/mobile/discrepancies` returns raw snake_case rows (`controller:362–369`) while the UI type expects `expectedPersons/actualPersons` (`person-discrepancy.api.ts:7–18`).
  5. The underlying table `person_discrepancies` **is never created by any migration** and is absent from `schema.prisma`; ratified as a known defect in `packages/db/migration-baseline.declarations.json:89–99` (`code-referenced-never-created`; "table absent from live").
  6. No component anywhere imports `PersonDiscrepancyPanel` or `MobilePersonDiscrepancyList` (repo-wide grep: only their own definitions + `PROJECT_STRUCTURE.md:2848-2849`); `apps/mobile` contains zero housekeeping API calls.
- **Operational consequence:** recording an HK person count or resolving a discrepancy cannot work against a migrated database; the discrepancy read always 404s even if the table existed.
- **Confidence:** HIGH (backend evidence additionally pre-committed by the repository's own declaration file).

### HK-004 [D] — Daily tag regeneration is destructive and cannot re-open a task; "daily" generation is structurally once-per-room-lifetime

- **Evidence:** `housekeeping.service.ts:162–207`. `generateDailyTasks()`:
  1. resets **every** non-guest task (`notes` not containing `'Guest:'`) of the hotel to `status:'pending'`, erasing `start_time`, `completed_at` — including `completed` and `inspected` tasks — **without writing any audit entry** for the erased transitions (`updateMany` path, `165–174`; audit only exists inside `updateTask`, `127–147`);
  2. then skips creation for any room that already has a task row in any state (`findFirst … { hotel_id, room_number }`, `183–185`) — there is no date predicate, because `housekeeping_tasks` has **no business_date column** (`schema.prisma:5759–5779`).
- **Consequences (all direct):** a room already assigned in `hk_room_assignments` gets created exactly once ever; after the first generation, pressing Generate again only resets yesterday's (or last month's) unfinished/approved work to `pending` and never bounds work to a day; re-assigning rooms to a different attendant does not propagate to existing tasks (assigned_to fixed at creation, `197`); approved-inspection history is unpinned from tasks while room-side status stays `INSPECTED` (rooms then show a pending task for an already-inspected room).
- **Confidence:** HIGH. (What is *intended* business semantics for "generate daily" is a Phase 2 decision; the mechanical behavior above is verified.)

### HK-005 [D] — Client-minted identifiers are persisted as database primary keys / business ids, with millisecond-collision exposure

- **Evidence:**
  - Server: `housekeeping.service.ts:59–60` `id: data.id || HK-${Date.now()}`, `task_id: data.task_id || data.id || HK-${Date.now()}`; `:223` assignment id `RA-${Date.now()}`; `:846` supervisor id `SUP-${Date.now()}`; generation `:189–191` `task_id: TASK-${Date.now()}` per room while `@@unique([hotel_id, task_id])` (`schema.prisma:5776`).
  - Clients mint these: `DailyAssignmentsSection.tsx:297` (`HK-${Date.now()}-${room}`), `:426` (`RA-${Date.now()}`), `GuestRequestsSection.tsx:146,379` (`HK-REQ-…`, `HK-${Date.now()}` — **no room suffix**), `:206` (`HK-${Date.now()}`), `InspectionsSection.tsx:176` (`SUP-${Date.now()}`), `LinenSection.tsx:470,820,912,941,1034,1312,1326` (`LT-`,`RLC-`,`BCH-`,`LTX-`,`LMV-`,`DAM-`,`LST-` series), `DashboardSection.tsx:293`.
- **Operational consequence:** two rapid-fire creations within the same millisecond (e.g. `handleStartAll`'s fire-and-forget loop, `DailyAssignmentsSection.tsx:351–357`, or multi-tab use) can collide on PK or `[hotel_id, task_id]`; the failures surface as raw 500s (create has no collision handling). Client-minted PKs also break server authority over id allocation.
- **Confidence:** HIGH for the mechanism (msec collision on `TASK-${Date.now()}` in the awaited generation loop is the most exposed); MEDIUM for observed frequency (needs runtime load to quantify).

### HK-006 [D] — Task completion does not update room condition; only inspection approval does, so an approved-only world is the single path by which rooms re-enter service

- **Evidence:** `updateTask`/`updateTaskStatus` paths write **only** `housekeeping_tasks` (+ audit) (`housekeeping.service.ts:83–150`); the complete transition (`117–119`) has no room SQL. The only room-condition writers in the entire API are checkout `DIRTY` (`check-out.handler.ts:471`) and approval `INSPECTED` (`housekeeping.service.ts:574–582`, both `rooms` and the `room_status` mirror via raw SQL).
- **Consequence:** a room cleaned but not yet inspected remains `DIRTY` at `rooms.housekeeping_status` and stays blocked for check-in (`room.aggregate.ts:30,90–99` allows only `CLEAN/INSPECTED/READY`); "completed" is invisible to front-desk readiness. This may be a deliberate gate — but its *mechanics* (no intermediate state between DIRTY and INSPECTED exists anywhere) are verified, and nothing implements a "CLEAN" stage (see HK-014).
- **Confidence:** HIGH.

### HK-007 [D] — HK module has zero role- or permission-gating; any authenticated tenant user can mutate any housekeeping state

- **Evidence:** exhaustive grep of `apps/api/src/modules/housekeeping` for `@Roles|@UseGuards|@Permission|@Permissions|SetMetadata`: only `@PropertyScope()` (`housekeeping.controller.ts:6,20`). Global guards no-op without metadata (`roles.guard.ts:10–14`, `permission.guard.ts:41–45` — `return true` when none declared). `POST /housekeeping/tasks` (`:41–44`, `createTask(@Body() body: any)`), `DELETE /housekeeping/room-assignments/:id` (`:273–276`), approve/reject inspection (`:285–293`), supervisor-assignment CRUD (`:317–335`) and both config PUTs (`:241–254`) are all open to any JWT holder within property scope. The platform registry defines `HOUSEKEEPING_MANAGER`/`HOUSEKEEPING_STAFF` roles (`platform/registry/permission-registry-init.service.ts:357–384`) but **no housekeeping route consumes any permission code** (contrast: Front Office guards its discrepancy routes, `front-office.controller.ts:44,51,61`).
- **Frontend:** no role/permission checks anywhere in `apps/web/app/(dashboard)/housekeeping/**` or the store (frontend track §6); `usePermissions` convenience codes (`read:housekeeping`) exist but gate nothing in this module.
- **Operational consequence:** privilege escalation within a tenant — a non-manager (any role string; `users.role` is a free string, `schema.prisma:13187`) can rewrite SLA config, delete assignments, approve inspections, create supervisor assignments.
- **Confidence:** HIGH. Severity: HIGH (authorization), per the audit's security mandate.

### HK-008 [D] — Inspection audit attribution is client-supplied and, in the live UI, literally empty

- **Evidence:** server takes `inspectorId/inspectorName` from the request body (`approveInspection/rejectInspection`, `housekeeping.service.ts:560,614`; controller passes raw body `:285–293`) and persists them into `room_inspections` comments (`:586–593`, `:639–646`) and `inspection_audit_log.performed_by/performed_by_name` (`:595–604`, `:662–672`). The authenticated context (`getRequestContext()?.userId`) is **never consulted** in these methods. The live UI hardcodes `''`: `InspectionsSection.tsx:342,365` (`handleApprove(approving,'','')`, `handleReject(rejecting,'','')`) → `parseInt('') → null`, audit rows carry `performed_by=''`, comments read `Approved by` (blank).
- **Consequence:** inspection history (`GET /inspections/history`, `InspectionsSection History tab`) records no usable inspector identity; the audit trail is both spoofable (any caller may name any inspector) and, in practice, blank.
- **Confidence:** HIGH. Severity: HIGH (audit integrity).

### HK-009 [D] — `generateDailyTasks` unique-key collision aborts generation mid-loop

- **Evidence:** loop `housekeeping.service.ts:180–205` issues `task_id: TASK-${Date.now()}` for each room; multiple rooms created within the same millisecond violate `@@unique([hotel_id, task_id])` (`schema.prisma:5776`); `prisma create` throws and the loop has no try/catch, so the controller 500s and remaining rooms are skipped.
- **Operational consequence:** on hotels with many rooms, daily generation proceeding to completion is probabilistic; a fast local DB round-trip (<1ms) makes collisions plausible.
- **Confidence:** HIGH for mechanism; MEDIUM for frequency (runtime-dependent).

### HK-010 [D] — Bulk discrepancy resolution reports success counts that are not verified against actual mutations

- **Evidence:** response `resolvedCount: command.data.discrepancyIds.length` (`bulk-resolve-discrepancies.handler.ts:56`) while the repository's bulk path never counts matched rows (`person-discrepancy.repository.ts:133–136`) and `AND hotel_id` predicates silently no-op foreign/unknown ids. `action` is not validated against `ResolutionAction` at the transport (`controller:400,419` `body.action as any`).
- **Consequence:** clients cannot rely on the success payload to know what happened; non-open/nonexistent ids resolve "successfully".
- **Confidence:** HIGH (currently anyway moot until HK-003's missing table is addressed — recorded so it does not get lost in remediation).

### HK-011 [D] — Supervisor assignment records overload `rooms` JSON with attendant ids, and reassignInspection writes a supervisor id over the attendant assignment

- **Evidence:** `getPendingInspections` reads supervisor rows' `rooms` arrays as **attendant id lists** (`housekeeping.service.ts:538–541`, `attendantIds.includes(attendantId)`); `createSupervisorAssignment` persists whatever the client sends (`:841–857`); `reassignInspection` sets `housekeeping_tasks.assigned_to = newSupervisorId` (`:814–817`) — overwriting the attendant who owns the task with a supervisor, never restoring it.
- **Consequence:** after any reassign call, pending-inspection grouping and task assignment semantics are corrupted for that task; nothing distinguishes `assigned_to` as attendant vs supervisor.
- **Confidence:** HIGH.

### HK-012 [D] — `getStaff()` joins `employees.employee_id` (business code) against `housekeeping_attendants.employee_id` (which the DB FK constrains to `employees.id` UUID)

- **Evidence:** service `:290–309` collects `attendants.map(a => a.employee_id)` and queries `employees.findMany({ where: { hotel_id, employee_id: { in: employeeIds } } })`. DB constraint: `housekeeping_attendants.employee_id → employees(id)` (`20261002000000_c03_late_reconciliation/migration.sql:1393`; schema declaration `:5708`). `employees.employee_id` is a separate column (`schema.prisma:3505`) with its own unique `[hotel_id, employee_id]`.
- **Consequence:** unless the two columns are identical in live data, `GET /housekeeping/staff` returns `[]` — and DailyAssignmentsSection's staff picker falls back to names derived from assignments/tasks (frontend track §9 fallback chain). Runtime behavior with live data is UNVERIFIED.
- **Confidence:** HIGH for the structural mismatch; the visible impact depends on live data equality.
- Related: **no writer exists for `housekeeping_attendants` anywhere** (`grep` = single read hit) — HK staff onboarding has no product path at all (HK-024).

## B. Design / architecture risks

### HK-013 [R] — Split-brain room condition fields across `rooms` and the legacy `room_status` mirror

- **Evidence:** two condition columns exist (`rooms.housekeeping_status` default `READY`, `room_status.housekeeping_status` nullable without default — `schema.prisma:10962` vs `11181`). Only HK approval writes the mirror (`housekeeping.service.ts:580`); checkout does not mirror `DIRTY` (`check-out.handler.ts:471`); no app reader of the mirror surfaced.
- **Risk:** state divergence between the two tables is guaranteed over time; any future reader of one vs the other disagrees.
- **Confidence:** HIGH (writers verified); MEDIUM for reader census completeness.

### HK-014 [R] — The canonical condition states CLEAN / INSPECTED-then-READY promised by config, shared constants, and FO docs have no production writer for CLEAN

- **Evidence:** `housekeeping_status` production writes = {`DIRTY` at checkout, `INSPECTED` at HK approval} only. `CLEAN` is a default of `front_office_status` (a dead column) and is otherwise only read (`room.repository.ts:56` fallback `housekeeping_status ?? front_office_status`). Shared transitions claim `CLEANING_IN_PROGRESS → CLEAN → INSPECTED` (`packages/shared/src/constants/housekeeping.constants.ts:161–164`); the status-flow config UI (service `:937–953`) and `room_status_config` (`10976–10988`, zero readers) encode an unfollowed order.
- **Risk:** the inspected-at-approval model leaves no recorded "cleaned" stage; SLA/inspection logic that assumes CLEAN cannot function; new work must choose between the shared vocabulary and the two real writers (Phase 2 decision, not resolved here).
- **Confidence:** HIGH.

### HK-015 [R] — `front_office_status` gates keep-out logic on a column nobody writes

- **Evidence:** `rooms.front_office_status` default `'CLEAN'`, never written by any workflow (grep; corroborated by `docs/front-office/PHASE_3_IMPLEMENTATION_REPORT.md:292,354`); yet it powers OUT_OF_SERVICE blocking (`room.aggregate.ts:36`, `room-assignment.service.ts:88,132`) and is stored as `hk_status` inside FO `room_discrepancies` (`get-discrepancies.handler.ts:60,70`).
- **Risk:** the OUT_OF_SERVICE gate is a permanent no-op in practice; discrepancy records carry a column value that cannot change.
- **Boundary note:** the columns are FO-owned; recorded here because HK readiness semantics read them.

### HK-016 [R] — Dual assignment paths disagree on housekeeping readiness: the walk-in/legacy assignment path allows DIRTY rooms

- **Evidence:** the check-in pipeline enforces readiness (`ASSIGNABLE_HOUSEKEEPING`, `room.aggregate.ts:30`, `isAssignableForCheckIn :90–99`; `RoomAssignableSpecification` raison `'ROOM_NOT_CLEAN'`, `room-assignable.specification.ts:36–61`), but the parallel walk-in path `front-office/domain/services/room-assignment.service.ts:55` declares `BLOCKED_HK_STATUSES = new Set<string>()` (empty) with a doc comment claiming the opposite ("Room not dirty unless explicitly allowed", `:43`), and `prisma-room-assignment.adapter.ts:23–33` has no HK clause.
- **Risk:** a DIRTY room can be walk-in-assigned while it would be rejected on the check-in path; the presence of two divergent gates is an architectural condition for conflict (documented, not fixed).
- **Confidence:** HIGH.

### HK-017 [R] — Run-time DDL and raw-SQL in the HK service layer (schema governance)

- **Evidence:** `ensureAuditLogTable()` creates the table at runtime (`housekeeping.service.ts:11–36`); supply-request service creates a sequence at runtime (`housekeeping-supply-request.service.ts:28`); person-discrepancy repo is all `$queryRawUnsafe/$executeRawUnsafe`, bypassing the Prisma tenant proxy (manually scoped, `person-discrepancy.repository.ts` throughout; proxy only covers model calls — `prisma.service.ts:279–295` per report). `requireHotelId()`-based manual scoping is what keeps raw SQL tenant-true.
- **Risk:** schema drift between migrations and runtime; scopes depend on discipline rather than a single enforcement point; ALS-absent contexts would pass unscoped raw SQL (proxy no-op behavior verified `prisma.service.ts:308–313`).
- **Confidence:** HIGH.

### HK-018 [R] — Config (`hk_sla_config`, `hk_status_flow_config`) is process-cached and write-failure-tolerant while claiming success

- **Evidence:** `getConfig` caches per `Map` in the service instance (`housekeeping.service.ts:880–901`) — never invalidated, divergent across replicas; `setConfig` catches DB failure and logs it (`914–917`); `updateSLAConfig/updateStatusFlowConfig` return `{ success: true }` regardless (`931–953`).
- **Consequence:** a dead `property_config` write still reports success; UI shows the new value until restart; per-instance cache means cluster nodes may serve different SLA/flow values.
- **Confidence:** HIGH.

### HK-019 [R] — OOO semantics are read-forked across five literal variants and `'OOO'` cannot pass the FO RoomStatus value object

- **Evidence:** readers split — `{OUT_OF_ORDER}` (availability adapter `:50`), `{OUT_OF_ORDER,OOO}` (aggregate `:33`), `+OOOS` (activities SQL), `{OOO,OUT_OF_ORDER}` (engineering provider `:24`), `{OOS,OUT_OF_SERVICE}` (`:47`); no production writer writes any OOO/OOS value anywhere (room-status track §6). FO `RoomStatus` whitelist lacks `'OOO'` → `RoomStatus.create('OOO')` throws, while the check-in repository excludes it and a spec round-trips it (`get-room-detail` spec `:240,250`).
- **Risk:** maintenance blocking (HK-relevant, see HK-020) exists only as UI badges; latent crash if an `OOO` row re-enters the FO aggregate path.
- **Confidence:** HIGH.

### HK-020 [R] — Out-of-order/out-of-service exists in vocabulary, UI, and a maintenance table but has no lifecycle owner

- **Evidence:** `room_maintenance` model exists (`schema.prisma:10546–10562`, status default `pending`), with an Engineering UI; engineering OOO/OOS counts read only (provider `:24,46–47`); no housekeeping, FO, or engineering code path writes `rooms.room_status` OOO/OOS or `front_office_status` OUT_OF_SERVICE (grep; only test seeds). HK room list (`getRooms`, service `:312–325`) and the entire task lifecycle are blind to maintenance blocks (task creation/checks never consult `room_maintenance`).
- **Consequence:** hotel "blocked room" workflows in the current product are informational only; HK task generation will include blocked rooms.
- **Confidence:** HIGH.

### HK-021 [R] — App-wide vocabulary fragmentation for task status/priority

- **Evidence:** DB tasks use lowercase (`pending | in-progress | completed | inspected | rejected | cancelled` — service `:117,357–362,571,630`); shared `TASK_STATUS` is UPPER (`housekeeping.constants.ts:34–43`); status-flow config stores UPPER (`:939–945`); priority defaults `'normal'` (DB) vs `'medium'` (store type `:42`) vs `LOW/MEDIUM/HIGH/URGENT` (shared `:59–64`) vs `LOW/NORMAL/HIGH/URGENT` (web front-office.constants `:96–101`) vs `LOW/MEDIUM/HIGH/CRITICAL` (`packages/shared constants/index.ts:45`).
- **Consequence:** cross-layer filters/maps are all value-shape-dependent; new consumers must translate; UI priority pickers may write values the backend never intended to filter by.
- **Confidence:** HIGH.

## C. UX / functional gaps

### HK-022 [G] — Dashboard tab statistics are client-synthesized with fake affordances

- **Evidence:** `DashboardSection.tsx` never calls `GET /housekeeping/dashboard` (endpoint exists, `controller:66–69`, `service:355–367` — zero callers per frontend track); stats derive from `roomAssignments × tasks` with per-(room,attendant) pseudo-task synthesis (`:199–249`, falsified `no-task-…` ids, SLA fallback 45); a permanent "AI Route Optimization Active" banner with no AI feature (`:334–337`); "Live updates" spinner is a 1-second re-render timer only (`:180–183`); in-progress bars animate to fabricated 45% (`:135–139`); a `<button>` "Reassign" with no onClick (`:122–124`).
- **Also:** store `dirtyRooms/cleanRooms` fields are never written anywhere (grep `429–430/547–548/1489–1490`), `frontDeskRequests` has no fetch action → the page.tsx unread badge (`page.tsx:50`) can never render, and "shift" in the staff panel is actually `s.department` (`:280`).
- **Confidence:** HIGH.

### HK-023 [G] — Six of the twelve `_components` files are dead code, one of them would crash if ever rendered

- **Evidence:** `TaskSheetsSection`, `SchedulingSection`, `MinibarSection`, `LostFoundSection`, `DeepCleanSection`, `ToastContainer` have zero importers repo-wide (grep verified; page imports only 6 sections, `page.tsx:7–12`). `MinibarSection.tsx:1` is `@ts-nocheck` and calls `fetchMinibarItems` from a store that has no such action (`TypeError` on mount). `DeepCleanSection` handlers are literal no-op stubs (`:21–28`). `SchedulingSection` "Confirm Assignment" only fires a toast (`:227–231`). `ToastContainer` duplicates the live `apps/web/components/ToastContainer.tsx` (imported at `apps/web/app/(dashboard)/layout.tsx:16,274`).
- **Confidence:** HIGH.

### HK-024 [R] — Housekeeping attendance/staff provisioning has no product write path

- **Evidence:** `housekeeping_attendants` is read in `getStaff()` only (single grep hit in `apps/api`); no endpoint, service, or script creates attendants; no migration seeds them (tests track §4: no `INSERT INTO housekeeping_*` seeds); `packages/cli/src/commands/db/seed.demo.ts` is a stub.
- **Consequence:** a fresh deployment cannot produce HK staff data through the product; the Add-Staff picker depends entirely on attendance rows created out-of-band.
- **Confidence:** HIGH.

### HK-025 [G] — No mobile or housekeeper-facing surface exists

- **Evidence:** `apps/mobile` staff tasks screen is a static mock (`app/(staff)/tasks/index.tsx:22` label only; no API calls). The `housekeeping/mobile/*` endpoints (controller `:357–387`) have no client at all. The React-Query feature (`features/housekeeping`) that would surface discrepancy panels is unmounted (HK-003.6).
- **Confidence:** HIGH.

## D. Inconsistencies

### HK-026 [I] — The same task-status transition matrix is enforced nowhere, audited everywhere, except supply requests

- **Evidence:** `updateTask` accepts any status change blindly (`:99`, mapped by value only) and merely *labels* transitions for the audit log (`mapStatusToAuditAction`, `:152–160`); no guard rejects e.g. `completed → pending` (used as "removed", `:157`) or `inspected → cancelled`. Contrast: supply-request service DOES enforce a real transition graph (`housekeeping-supply-request.service.ts:6–13,43–49`) — proving the codebase's own standard.
- **Consequence:** task lifecycle integrity depends entirely on UI behavior; API consumers can drive any-to-any transitions including inspected→pending.
- **Confidence:** HIGH.

### HK-027 [I] — Readiness acceptability sets disagree by consumer

- **Evidence:** check-in accepts `CLEAN/INSPECTED/READY` (`room.aggregate.ts:30`); scheduled room move accepts only `CLEAN/INSPECTED` and **rejects the schema default READY** (`execute-scheduled-room-move.handler.ts:61–68`); reservations and group allotments ignore housekeeping entirely (sellability = `room_status='AVAILABLE'`, `prisma-inventory-reservation.adapter.ts:33–39`; verified F-25 comment); the legacy assignment service accepts everything (`:55`); Day-use towels `AVAILABLE/READY` only per web helper.
- **Consequence:** the question "is this room ready?" has four different answers depending on the code path; HK approval (INSPECTED) is sufficient everywhere, but READY (the default) fails one consumer and passes another.
- **Confidence:** HIGH.

### HK-028 [I] — Frontend fabricates room condition where the new check-in policy forbids fabrication

- **Evidence:** `DailyAssignmentsSection.tsx:147,391` renders missing `housekeeping_status` as `'CLEAN'`; `DashboardSection` synthesizes default room types and SLA values (`:62,229`). The FO check-in side was explicitly rebuilt to treat unknown readiness as NOT ready ("never assume a room is ready because a status field is absent", `room.aggregate.ts:97–99` comment; web worklist specs "never fabricates readiness").
- **Consequence:** within one property state, HK UI displays invented cleanliness; FO UI displays absence. Which one is authoritative is a Phase 2 decision.
- **Confidence:** HIGH.

### HK-029 [I] — Frontend/room rack mixes `room.status` with `housekeepingStatus` (FO rack filter bug) and overlaps HK badge vocabulary

- **Evidence:** `RoomRack.tsx:25` compares filter against `room.status` not `housekeepingStatus` (documented as FO-P1-015 in `docs/front-office/PHASE_1_GAP_REGISTER.md:208`, still live); `RoomCard.tsx:9,18,51` includes `OCCUPIED/AVAILABLE` inside `HOUSEKEEPING_STATUSES`; FO rack rows derive `housekeepingStatus` from `rooms.room_status` (`front-office.query.handlers.ts:462–470`, comment `:322`).
- **Boundary note:** the defect sites are FO-owned; registered because the audit mandate covers housekeeping-status semantics wherever they are consumed.
- **Confidence:** HIGH.

### HK-030 [I] — The guest-request coupling is a string protocol between two UIs, with its own corrupted-history side effect

- **Evidence:** DailyAssignments creates HK tasks from room-service requests and encodes the link in task `notes` as ``Guest: <type> - <note> [<requestId>]`` (`DailyAssignmentsSection.tsx:211–219,243–250,60–65`, regexes); generation + pending-inspection queries branch on the literal `'Guest:'` (`housekeeping.service.ts:166–172,505–513` — any future note legitimately containing "Guest:" changes system behavior); completion of the task best-effort completes the request (`:324–342`, fire-and-forget); cancellation from GuestRequests PATCHes the task to `'cancelled'` directly (`:808–814`). `rejectInspection` similarly parses/cleans `notes` with regex `/\|\| Rejected: .+?$/` (`housekeeping.service.ts:623–626`).
- **Consequence:** data semantics live in free text; collision with the HK-004 reset filter is real (a guest-request task is *excluded* from daily reset, by note sniffing).
- **Confidence:** HIGH.

### HK-031 [I] — Staff "shift" concept does not exist in the live module

- **Evidence:** no shift field anywhere on tasks/assignments/models (`hk_room_assignments` model read; task columns read; store interfaces read); the only shift UI is the dead `SchedulingSection` (fake `M/—` grid, `:146–149,10`) and HR payroll mock data (`apps/web/app(hr-payroll)/page.tsx:19–41`). `type` on assignments (`room | corridor | … + supervisor`) is the only grouping concept.
- **Position vs. the Phase-0 brief "day shift / night shift":** UNVERIFIED / NOT IMPLEMENTED — recorded as an open business question for Phase 2 (Q-item HK-036).
- **Confidence:** HIGH that no shift exists in code; business intent unknown.

## E. Enhancement candidates (observations only — not proposals for implementation)

- **HK-032 [E]** — `POST /housekeeping/inspections/:id/reassign` exists but no UI uses it (`controller:312–315`, `service:806–833`); its purpose is ambiguous with the ✕ "send back to attendant" flow (InspectionsSection `:208–217`). Candidates for Phase 2 consolidation or removal decision.
- **HK-033 [E]** — `POST /housekeeping/inspections/audit-log/history` — history tab is not refetched after approve/reject (store refetches pending+tasks+audit only, store `:894–909`) → stale until re-entry. Small-data-refresh candidate.
- **HK-034 [E]** — `GET /housekeeping/dashboard` endpoint exists unused; the Dashboard tab could either consume it (server-authoritative counters incl. `inspected/rejected`) or the endpoint could be a Phase-2 deletion candidate — not to be decided in Phase 1.
- **HK-035 [E]** — linen "location split" synthesis (service `:472–477` fixed ratio 60/25/10/5) vs the seeded real `linen_inventory` par data and `packages/cli`'s seed.linen — a real per-location model exists in spirit (seed par-per-room via `rooms.groupBy`) yet the API only synthesizes. Tech-inventory alignment is ripe for Phase 2.
- **HK-036 [Q]** — Business date, shift, workload/credit formulas, capacity, cadence rules: none verifiable in code — all are Phase 2 business decisions (register Q+ below).

## F. Late-verification addition (confirmed while producing Deliverable 5)

- **HK-037 [D]** — Supply-request approval is unreachable: `approve()` validates its own transition map, which grants no SUBMITTED→APPROVED edge, and no code path ever sets `PENDING_APPROVAL`, the only state from which APPROVED is legal.
  - Evidence: `VALID_TRANSITIONS` at `apps/api/src/modules/housekeeping/housekeeping-supply-request.service.ts:5–14` — `SUBMITTED: ['PENDING_APPROVAL', 'CANCELLED']`, `PENDING_APPROVAL: ['APPROVED', …]`; `validateTransition` at `:43–49` throws `BadRequestException`; `approve()` at `:379–405` calls `validateTransition(detail.status, 'APPROVED')`. Repo-wide grep of the file shows `PENDING_APPROVAL` literals only at `:7`, `:8` (the map) and `:147–148` (dashboard `COUNT(*) FILTER` display counters) — no writer. `submit()` sets `SUBMITTED` directly at `:371–372`.
  - Consequence: every submitted request terminates at SUBMITTED; `/housekeeping/supply-requests/:id/approve` can only ever 400. Consistent with 03_WORKFLOW_MAPS.md §8 ("approval is dead") — now formally registered. Classification [D]: violates the module's own state contract (an approved state is offered, listed in the transition map at line 9 as a source of PARTIALLY_FULFILLED, yet cannot be entered from any reachable flow). Confidence HIGH (single-file, fully closed system; direct read). Severity MEDIUM (feature is backend-orphaned anyway — no UI calls `/housekeeping/supply-requests`, see Annex A — so no live user impact; impact applies to any API consumer).
  - Related: HK-026 (transition matrices enforced nowhere else / here), Phase-2 question of intended flow (auto-advance SUBMITTED→PENDING_APPROVAL on submit? multi-approval semantics at `:383–388`).

## Annex A — endpoint contract cross-reference (condensed)

Full mapping produced by the contract track; both sides read in full.

| Live frontend → existing backend (✓) | Dead frontend → missing backend (✗) |
|---|---|
| GET/POST `/housekeeping/tasks`; PATCH `/housekeeping/tasks/:id/status`; PUT `/housekeeping/tasks/:id/assign` (route ✓, caller ✗ — `assignTask` never invoked: `Dashboard`'s only reassign <button> has no handler, `DailyAssignments` uses create/update instead) | posting `/housekeeping/requests…` (3), `/housekeeping/teams`, all `/housekeeping/linen/*` writes, all `/housekeeping/laundry/*`, `/housekeeping/supplies/*`, `/housekeeping/room-assignments/move-room`, `POST /lost-found/items/:id/claim`, `GET /housekeeping/discrepancies` |
| POST `/housekeeping/tasks/generate-daily`; room-assignments + supervisor-assignments CRUD; `config/{sla,status-flow}` GET/PUT | — |
| GET `/housekeeping/staff|rooms|sections|outlets|banquet-rooms`; `/housekeeping/inspections/{pending,approve,reject,audit-log,history}` | — |
| GET `/housekeeping/linen`, `/linen/types`, `/linen/locations`, `/linen/dashboard` (reads only; GET `/linen/locations` returns synthesized splits) | — |
| Backend-only (`✓` route, no UI caller): GET `/housekeeping/dashboard`; POST `inspections/:id/reassign` | Frontend-only (`✗`): see dead column above |
| Cross-module ✓: `/front-office/due-outs` (DailyAssignments `:132`), `/room-service/requests*` (GuestRequests + DailyAssignments), `/front-office/discrepancies*` (FO UI, permission-gated) | person-count/resolve payloads mismatched (HK-003) |

## Annex B — findings↔evidence-classification cross-walk (Phase 1 deliverable 5 cross-ref)

- Runtime-unexecuted items relying on live-DB presence: HK-003 (table absent per ratified declaration), HK-012 (staff join equality), HK-006 (collision frequency).
- Runtime-unexecuted UI behavior claims (render-stream logic only): HK-001 move-room UX, HK-022.
- Everything else carries direct source line references verified in this audit.
- HK-037: single-file closed system, direct read; no runtime verification performed (and none required absent a DB-resident SUBMITTED row — the guard is in code, before persistence).

# 13 — Reservation Operations: Mews Parity Audit

**Type:** Read-only audit (no implementation)
**Date:** 2026-10-15
**Scope:** XYLO Reservation Operations (Reservations, Front Office, Group/Allotment, Billing/Folio, Availability) assessed against publicly documented **Mews Operations** capabilities.
**Status:** For review. No remediation performed. Awaiting approval before any change.

> **Read-only guarantee.** This audit made **no** code, schema, migration, database, deployment, or configuration changes. No database was mutated. `xylo_cloud` was not touched. `xylo_test` was **not** used as production-schema evidence. The only artifact produced is this Markdown document.

---

## 0. Method, Evidence Convention, and Known Limitations

### 0.1 Evidence tags

Every claim in this report carries an evidence tag:

| Tag | Meaning |
|-----|---------|
| **[V]** | **Verified directly** by this audit — file/line inspected or a targeted command run during this audit. |
| **[R]** | **Reported** by a parallel audit stream — returned as a finding with cited file:line evidence but **not independently re-executed** by this audit. Treated as a lead, not a proof. |
| **[M]** | **Mews documented behavior** — sourced from an official Mews surface (Help Center article metadata/snippet, Mews product page, or `docs.mews.com` Connector API reference). |
| **[U]** | **Unverifiable** — the Mews behavior could not be confirmed from an official, machine-retrievable source. Classified as UNRESOLVED downstream, not guessed. |

### 0.2 Mews source-retrieval limitation (important)

The **Mews Help Center** (`help.mews.com`) is a JavaScript single-page application (Salesforce Experience Cloud). Every available retrieval path — direct fetch, `r.jina.ai` text extraction, and the Wayback Machine — returns a shell page (`"CSS Error"` / `"Loading…"`) rather than article bodies. Consequently:

- Mews behavior in this report is sourced from **official article metadata/snippets** (search-engine indexed descriptions of official Mews articles), the **official Mews product page**, and the **official Mews Connector API reference** (`docs.mews.com`).
- Where a behavior could only be inferred from a snippet or from marketing copy, it is tagged **[U]** and the capability is marked **UNRESOLVED** rather than asserted.
- No Mews behavior in this report is inferred from screenshots or marketing alone. This limitation is carried forward into the Verification Backlog (§6) as a first-class task.

### 0.3 Classification legend

Each in-scope capability is assigned **exactly one**:

`IMPLEMENTED AND VERIFIED` · `IMPLEMENTED—VERIFICATION INCOMPLETE` · `PARTIALLY IMPLEMENTED` · `IMPLEMENTED BUT DEFECTIVE` · `MISSING` · `NOT APPLICABLE` · `UNRESOLVED`

Rules applied:
- A capability is **not** classified MISSING solely because it was not found — MISSING requires either (a) a definitive negative search across the codebase or (b) a positive confirmation that no route/handler exists.
- A capability is **not** classified VERIFIED solely because a test file exists. Where "verified," this report states **what the test actually proves**.
- Known-good XYLO work (arrival/departure worklists, date filters, search by guest name, status/date/room-type filters, room assignment, room moves/upgrades/overbooking, reservation detail, FO arrival/departure + room readiness, Groups/Rooming Lists/Allotments, reservation↔folio + guest/company separation) is **preserved** unless contradicted by evidence. Contradictions are recorded as **evidence discrepancies** (§3.7, §4.1).

### 0.4 Scope boundaries respected

- Reused existing audit/test/closed-phase evidence; did not redo completed work.
- Preserved roadmap order, ADRs, and closed phases. Phase 2b / 6 / 7 remain NOT STARTED; Phase 9 remains PARTIAL. **No closed phase was reopened.**
- No new phase was invented; this report is a cross-cutting audit artifact, not a roadmap item.

---

## 1. Executive Summary

XYLO's Reservation Operations surface is **broad and functionally deep**, but its depth is **uneven and its assurance is thin in exactly the places enterprise hotel operations are most exposed**: concurrency, financial controls, authorization on mutation routes, and query precision.

The platform demonstrably **matches Mews on the core operational loop**: reservation discovery worklists, date/status/room-type filtering, arrival and departure worklists, reservation detail, front-desk arrival/departure with room-readiness, room assignment, room moves, upgrades, overbooking limits, groups/blocks/allotments, and reservation↔folio coupling with a group master folio for separating guest vs. company charges. These are real and largely verified.

However, the audit surfaces **one structural gap, one financial-control gap, and one authorization gap** that materially separate XYLO from Mews' documented posture:

1. **Concurrency safety is nominal, not real.** A `ConcurrencyConflictError` type exists but is **never thrown**; reservation writes are **last-write-wins** with no optimistic locking/version column, and there is **no overlap/exclusion constraint** protecting a room from double-assignment under concurrent requests. Mews explicitly prevents assigning a space to a reservation when it is already occupied by another reservation block. XYLO's overlap detection is **computed for display** (the check-in room-option list) but **not enforced at the assignment write**. *(§2.C, §5 D-1)*

2. **Check-out does not enforce a balanced folio.** Mews documents checking out "with unbalanced bill" as an **explicit, deliberate** action — implying the default is that an unbalanced bill **blocks** check-out. XYLO's `closeFolio()` simply flips `status = 'CLOSED'` with no balance gate, and the check-out handler calls it inside a `try/catch` that **logs and continues** on failure. The financial gate exists in the UI only. *(§2.E, §5 D-2)*

3. **Authorization is present on most mutation routes but absent on the highest-risk few.** `walk-in`, `check-out`, and `reverse-checkin` on the Front Office controller carry **no permission decorator**, and the controller opts out of property scoping via `@PropertyScope(false)`. Walk-in additionally **does not actually check anyone in** — it creates a `CONFIRMED` reservation and stops. *(§2.C, §5 D-3)*

Additionally, structured **search is under-implemented**: the `SearchReservationDto` advertises `confirmationNo`, `guestName`, `roomNumber`, `companyName`, `agentName`, `marketCode`, `sourceCode`, and `reservationType`, but the repository consumes only a single free-text `search` and matches **guest full name, confirmation number, and reservation id only** — **room number is not searchable at all**, and company/agent/market/source/type parameters are silently ignored. **Saved views / saved filters are entirely absent.** *(§2.A, §5 D-4/D-5)*

**Final verdict: PARTIAL PARITY WITH MATERIAL GAPS** — the operational surface is substantially present and much of it is verified, but concurrency, financial-control, and authorization gaps preclude an enterprise-parity claim until remediated and independently verified. (Full rationale in §8.)

---

## 2. Mews Feature Inventory

Each row records the **Mews capability**, the **documented behavior**, the **source**, and the source tag. Sources marked **[U]** are snippet/metadata-derived; see §0.2.

### 2.A Discovery, Worklists, Search, Reservations List

| # | Mews capability | Documented behavior | Source | Tag |
|---|-----------------|---------------------|--------|-----|
| A1 | Reservation list / Reservations module | Central list of all reservations with columns and per-reservation actions | Mews Help Center — Reservations module overview (article metadata/snippet) | [M]/[U] |
| A2 | Reservation states | Inquired, Optional, Requested, Confirmed, Started, Processed, Canceled | Mews Help Center — Reservation states (article metadata) | [M] |
| A3 | Reservation origins | Distributor, Channel manager, Commander, Import, Connector, Navigator; Commander origins include In person, Channel, Phone, Email, Website, Message, Call center, Rooming list | Mews Help Center — Reservation origins (article metadata) | [M] |
| A4 | Get all reservations (filters) | Filtering by state, dates, space category, rate, group, etc. | `docs.mews.com` Connector API — `Get all reservations` | [M] |
| A5 | Global search | Dashboard search bar; prioritizes results by stay dates and guest status; supports shortcuts | Mews Help Center — Search (article metadata/snippet) | [M]/[U] |
| A6 | Dates / status / room-type filtering | Filter reservations by arrival/departure window, state, and space category | Mews Help Center / Connector API filters | [M] |
| A7 | Saved views / saved filters | Persistable filtered list configurations | Mews Help Center — Saved filters/views (article metadata) | [M]/[U] |
| A8 | Reservation detail | Full reservation profile: guest, stay, rate, space, notes, history | Mews Help Center — Reservation detail (article metadata) | [M]/[U] |

### 2.B Reservation Lifecycle (write operations / state machine)

| # | Mews capability | Documented behavior | Source | Tag |
|---|-----------------|---------------------|--------|-----|
| B1 | Create / Confirm / Cancel / Reinstate | Full reservation lifecycle transitions | Mews Help Center — states + Connector API `Add reservation` / `Update reservation` | [M] |
| B2 | No-show / Optional expiry / Requested handling | State-driven handling of non-arrivals and pending states | Mews Help Center — Reservation states | [M] |
| B3 | Rate change / space category change | Change rate or space category on a reservation | Mews Help Center — Reservation module features (article metadata) | [M]/[U] |
| B4 | Extend stay (shorten/lengthen) | Modify departure date | Mews Help Center — extend/amend reservation | [M]/[U] |
| B5 | Guarantee / deposits | Guarantee type and deposit handling | Mews Help Center — deposits (article metadata) | [M]/[U] |
| B6 | Group merge | Merge group reservations | Mews Help Center — reservation module features | [M]/[U] |
| B7 | Activity/event change audit | Reservation history / change log | Mews Help Center — Reservation history (article metadata) | [M]/[U] |

### 2.C Front Desk Operations

| # | Mews capability | Documented behavior | Source | Tag |
|---|-----------------|---------------------|--------|-----|
| C1 | Check-in | Space must be **Inspected** before check-in is allowed | Mews Help Center — Check-in (article metadata) | [M] |
| C2 | Blocked space | A **blocked** space prevents check-in | Mews Help Center — Check-in (article metadata) | [M] |
| C3 | Bulk group check-in | Check in an entire group with a single action | Mews Help Center — Check-in (article metadata) | [M] |
| C4 | Room assignment | Assign a space; Mews prevents assigning a space already consumed by another reservation block | Mews Help Center — space assignment / availability (article metadata) | [M] |
| C5 | Room move / upgrade | Move a guest between spaces; upgrade to a higher category | Mews Help Center — moving/upgrading reservations | [M]/[U] |
| C6 | Overbooking | Controlled oversell against a configured allowance | Mews Help Center — overbooking (article metadata) | [M]/[U] |
| C7 | Check-out | Close **all open bills**, or explicitly **"Check out with unbalanced bill"** | Mews Help Center — Check-out (article metadata) | [M] |
| C8 | Reverse check-in | Undo a check-in | Mews Help Center — reverse check-in (article metadata) | [M]/[U] |
| C9 | Walk-in | Create + check in a guest at the desk without prior reservation | Mews Help Center — walk-in (article metadata) | [M]/[U] |
| C10 | Timeline | Visual space-allocation grid; optimize allocation; create reservation from Timeline | Mews Help Center — Timeline (article metadata) | [M]/[U] |
| C11 | Room readiness / housekeeping status | Inspected / dirty / out-of-order status feeding check-in eligibility | Mews Help Center — housekeeping / check-in | [M]/[U] |

### 2.D Groups, Blocks, Allotments, Rooming Lists

| # | Mews capability | Documented behavior | Source | Tag |
|---|-----------------|---------------------|--------|-----|
| D1 | Group reservations | Groups of reservations under one master record | Mews Help Center — Group reservations | [M] |
| D2 | Blocks / allotments with release dates | Inventory blocks with a release (cut-off) date | Mews product page — group allocations / block release dates | [M] |
| D3 | Rooming list | Manage/import a list of names-to-rooms for a group | Mews Help Center — Rooming lists (article metadata) | [M] |
| D4 | Rooming list import/export | Import/export rooming lists (file-based) | Mews Help Center — import rooming list | [M]/[U] |
| D5 | Group size cap | Group reservation cap documented (e.g., 750 reservations per group) | Mews Help Center — group limits (article metadata) | [M]/[U] |
| D6 | Allotment wash / release to house | Unpicked allocation washing back to general availability at cut-off | Mews Help Center — allotments / wash | [M]/[U] |
| D7 | Master (group/company) folio | Charges routed to a company/master folio, separating guest vs. company | Mews Help Center — company billing / routing | [M]/[U] |

### 2.E Billing, Folio, Deposits

| # | Mews capability | Documented behavior | Source | Tag |
|---|-----------------|---------------------|--------|-----|
| E1 | Guest folios + windows | Multiple folio windows per reservation (routing) | Mews Help Center — folios / billing | [M]/[U] |
| E2 | Company / Travel-agent folio separation | Direct charges to a company or travel-agent account | Mews Help Center — company billing | [M]/[U] |
| E3 | Deposits / unallocated payments | Deposits taken pre-arrival and applied to the folio across the lifecycle | Mews Help Center — deposits | [M] |
| E4 | Balanced-bill check-out gate | All open bills must be closed (or explicit override) before check-out | Mews Help Center — Check-out | [M] |
| E5 | Financial posting traceability | Every charge/payment traceable to an actor and source; no duplicate posting | Mews Help Center — billing / audit trail | [M]/[U] |
| E6 | Refunds / credits | Refund processing against posted charges | Mews Help Center — refunds | [M]/[U] |

### 2.F Availability, Rates, Enterprise Controls

| # | Mews capability | Documented behavior | Source | Tag |
|---|-----------------|---------------------|--------|-----|
| F1 | Live availability | Real-time availability across space categories and channels | Mews product page — live availability | [M] |
| F2 | Rate management | Rate plans, restrictions, and rate codes | Mews product page — rate management | [M] |
| F3 | Group allocations | Allocated inventory per group with release dates | Mews product page — group allocations | [M] |
| F4 | Multi-property / enterprise scoping | Property-scoped access to reservations and financial data | `docs.mews.com` Connector API (per-enterprise/property tokens) | [M] |
| F5 | Financial idempotency / duplicate-prevention | Posted transactions are not duplicated on retry | Mews Help Center — billing integrity (article metadata) | [M]/[U] |
| F6 | Cross-property reservation transfer | Moving a reservation between properties | Mews Help Center — transfer (article metadata) | [M]/[U] |

---

## 3. XYLO Parity Matrix

Legend per §0.3. Evidence tags per §0.1.

### 3.A Discovery, Worklists, Search, Reservations List

| Mews | XYLO capability | Evidence | Classification |
|------|-----------------|----------|----------------|
| A1 | All-Reservations list + detail endpoints | `reservations.controller.ts` (59 routes incl. list, detail, views) **[V]**; Phase 9 All-Reservations table re-wire still pending per roadmap **[V]** | PARTIALLY IMPLEMENTED |
| A2 | Reservation status vocabulary | ADR-077 status vocabulary; states modeled; commands implemented **[V]**. Mews also models Optional/Requested/Inquired — XYLO coverage of the full state set not fully confirmed **[R]** | PARTIALLY IMPLEMENTED |
| A3 | Reservation origins | `SearchReservationDto` declares `sourceCode`/`marketCode`; origin model exists via source/market codes **[V]**; full Mews origin taxonomy (Commander origins etc.) not modeled as first-class **[R]** | PARTIALLY IMPLEMENTED |
| A4 | Filter by date/status/room-type | Date filters, status filters, room-type filters present in repository `findAll` **[V]** (`arrival_date`, `room_type`) and quick date filter **[V]** (`quickDateFilterBounds`) | IMPLEMENTED AND VERIFIED |
| A5 | Global/guest search | Free-text `search` only; matches **guest full name, confirmation_no, reservation id** (`reservation.repository.ts:143-157`) **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| A6 | Status/date/room-type filters | Confirmed present **[V]** | IMPLEMENTED AND VERIFIED |
| A7 | Saved views / saved filters | **0 matches** for saved-view/saved-filter across `apps` **[V]** | MISSING |
| A8 | Reservation detail | Detail endpoint + UI present **[V]**; roadmap notes detail re-wire work pending | PARTIALLY IMPLEMENTED |

**Evidence discrepancy (A5 / search-by-room-number):** The task's "preserve known work" list includes *search by room number*. Direct evidence contradicts a *functional* room-number search: the repository search clause (`:152-157`) matches `guest_id`, `confirmation_no`, and `id` — **not** room number. `SearchReservationDto:16-23` **declares** `roomNumber` but the repository never reads it. Recorded as **evidence discrepancy ED-1** (§3.7): room-number *parameter plumbing* may have existed historically, but the executed query does not search by room number today.

### 3.B Reservation Lifecycle

| Mews | XYLO capability | Evidence | Classification |
|------|-----------------|----------|----------------|
| B1 | Create/Confirm/Cancel/Reinstate | Commands implemented (roadmap command list) **[V]** | IMPLEMENTED AND VERIFIED |
| B2 | No-show / Optional / Requested | No-show command exists; full Optional/Requested lifecycle not confirmed **[R]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| B3 | Rate change / room-type change | `ChangeRate`, `ChangeRoomType` commands implemented **[V]** | IMPLEMENTED AND VERIFIED |
| B4 | Extend | `Extend` command + `POST :id/extend` **[V]** | IMPLEMENTED AND VERIFIED |
| B5 | Guarantee / deposit | `Guarantee`, `PostDeposit`, `TransferDeposit` commands implemented **[V]** | IMPLEMENTED AND VERIFIED |
| B6 | Group merge | Not confirmed in XYLO **[R]** | UNRESOLVED |
| B7 | Lifecycle change audit | Domain events published (7 events, Phase 3) **[V]**; durable per-command journal **not** used by reservation command handlers **[V]** | PARTIALLY IMPLEMENTED |

### 3.C Front Desk Operations

| Mews | XYLO capability | Evidence | Classification |
|------|-----------------|----------|----------------|
| C1 | Check-in requires Inspected space | Room-readiness + eligibility logic present (`front-office/check-in/`) **[R]/[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| C2 | Blocked space blocks check-in | Blocking modeled; check-in eligibility checks hold/block **[R]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| C3 | Bulk group check-in | Batch status command exists (`BatchUpdateStatus`); true bulk group check-in not confirmed **[R]** | UNRESOLVED |
| C4 | Room assignment + anti-double-assignment | Assignment path present; **`assignedToOtherReservation` overlap flag is hydrated for the check-in room-option list but NOT supplied by the assign-room write path**; no DB `EXCLUDE`/overlap constraint **[V]** | IMPLEMENTED BUT DEFECTIVE |
| C5 | Room move / upgrade | Transfer-room + upgrade-room handlers converged to repo methods (ADR-078) **[R]/[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| C6 | Overbooking | `overbooking_limits` table read path (A1 adapter); `t42-overbooking-guard.spec.ts` pins posture **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| C7 | Check-out balanced-bill gate | `closeFolio()` has **no balance gate** (`billing.service.ts:878-896`); check-out handler calls it in `try/catch` that **logs and continues** (`check-out.handler.ts:525-530`) **[V]** | IMPLEMENTED BUT DEFECTIVE |
| C8 | Reverse check-in | Handler + route exist (`front-office.controller.ts:95-98`) **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| C9 | Walk-in | Handler creates a `CONFIRMED` reservation **only** — no check-in, no room assignment (`sub-resources.handlers.ts:471-490`) **[V]**; ADR-073 defines walk-in as a Front Office entry point | IMPLEMENTED BUT DEFECTIVE |
| C10 | Timeline | Not confirmed as a Mews-style space-allocation grid; front-office room-grid exists (`getRoomGrid`) **[V]** | UNRESOLVED |
| C11 | Room readiness / housekeeping | Room-readiness integrated into FO arrival/departure **[R]/[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |

### 3.D Groups, Blocks, Allotments, Rooming Lists

| Mews | XYLO capability | Evidence | Classification |
|------|-----------------|----------|----------------|
| D1 | Group reservations | `group-allotment` module with group-booking commands/controllers **[V]** | IMPLEMENTED AND VERIFIED |
| D2 | Blocks/allotments + release dates | Allotment controller with release/wash endpoint ("Release allotment allocation (wash to house)") **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| D3 | Rooming list | **No `roomingList`/`rooming list` implementation anywhere in `apps/api/src/modules`** (0 matches) **[V]** | MISSING |
| D4 | Rooming list import/export | Same as D3 — no implementation; any UI button is a dead control **[V]** | MISSING |
| D5 | Group size cap | Not confirmed in XYLO **[R]** | UNRESOLVED |
| D6 | Allotment wash | `ExecuteCutOffWashHandler` + wash-time attrition assessment **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| D7 | Master (company) folio | `GetMasterFolio`, `PostMasterCharge`, master-folio charge/payment/reconcile endpoints on `group-booking.controller.ts:126-162` **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |

**Note on D3/D4 (evidence discrepancy ED-2):** The task's preserve-list includes *Rooming Lists*. Direct evidence shows a **group/allotment** capability exists, but a **rooming-list** (name-to-room list with import/export) is **not implemented** in the backend. The two are distinct Mews capabilities (D1/D2 vs. D3/D4). Recorded as **ED-2**.

### 3.E Billing, Folio, Deposits

| Mews | XYLO capability | Evidence | Classification |
|------|-----------------|----------|----------------|
| E1 | Guest folios + windows | `folios` (PK text, `reservation_id`, `window_no`) + `folio_postings`; `getFolioReferences` / `settleFolioBalance` per window **[V]** | IMPLEMENTED AND VERIFIED |
| E2 | Company / TA folio separation | Reservation→folio coupling + **group master folio** for company charges **[V]**; no separate **AR/company account** object in cashiering (routing is to `window_no` on the same reservation) **[V]** | PARTIALLY IMPLEMENTED |
| E3 | Deposits / unallocated payments | `reservation_payment` legacy rows + deposit path; `transferDepositToFolio` uses a `${hotelId}:${reservationId}` marker for dedupe **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| E4 | Balanced-bill check-out gate | **Not enforced server-side** (see C7) **[V]** | IMPLEMENTED BUT DEFECTIVE |
| E5 | Posting traceability / duplicate-prevention | **Billing module** has `idempotencyKey` + `externalReference` + folio advisory lock + in-transaction idempotency re-check (`billing.service.ts:127-165`; DTO `:40,88,118`) **[V]**; **reservations `prisma-billing.adapter.ts` write path does NOT use idempotency** (marker-string dedupe only) **[V]** | PARTIALLY IMPLEMENTED |
| E6 | Refunds / credits | `processRefund` + `postCredit` present with idempotency support (**[V]** `billing.dto.ts:88,118`) | IMPLEMENTED—VERIFICATION INCOMPLETE |

### 3.F Availability, Rates, Enterprise Controls

| Mews | XYLO capability | Evidence | Classification |
|------|-----------------|----------|----------------|
| F1 | Live availability | Availability engine + assertion journal **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| F2 | Rate management | Rate-change command; rates-inventory integration is a **Phase 6 NOT STARTED** target **[V]** | PARTIALLY IMPLEMENTED |
| F3 | Group allocations | `group-allotment` module **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| F4 | Multi-property / property scoping | `PropertyScopeGuard` (APP_GUARD) + AsyncLocalStorage repository scoping **[V]**; **reservations RLS is enabled but policy-less** (only one `CREATE POLICY` in `packages/db`, on `tax_codes`) **[V]** | IMPLEMENTED—VERIFICATION INCOMPLETE |
| F5 | Financial idempotency | See E5 — present in billing module, **absent** on the reservations adapter path **[V]** | PARTIALLY IMPLEMENTED |
| F6 | Cross-property transfer | Deferred by ADR-072 (requires multi-property authorization not yet implemented) **[V]** | NOT APPLICABLE (deferred by ADR) |

### 3.7 Evidence discrepancies register

| ID | Discrepancy | Preserve-list claim | Direct evidence | Resolution |
|----|-------------|---------------------|-----------------|------------|
| ED-1 | Room-number search | "search by … room number" is known-good | Repository search matches guest name/confirmation/id only (`reservation.repository.ts:143-157`); `roomNumber` param declared but unused **[V]** | Capability **partially** true; room-number search **not functional** today |
| ED-2 | Rooming lists | "Groups / Rooming Lists / Allotments" preserved | Groups + allotments present; **no rooming-list backend** (0 matches) **[V]** | Split: D1/D2 preserved; D3/D4 MISSING |
| ED-3 | Overbooking | "overbooking" preserved as working | Overbooking is a **limits read** + guard spec; no anti-double-assign constraint **[V]** | Present as allowance config; **not** a hard allocation guarantee |
| ED-4 | Guest/company folio separation | "guest/company separation" preserved | Group **master folio** supports company charges **[V]**; no separate AR/company **account** object; cashiering routes only to `window_no` **[V]** | Separation exists via master folio; AR-account model absent |

---

## 4. Confirmed Gap Register

Gaps are **confirmed absent** (positive negative-search or positive confirmation of non-existence), not merely "not found."

| ID | Gap | Mews counterpart | Severity | Evidence |
|----|-----|------------------|----------|----------|
| G-1 | **Saved views / saved filters** — none | A7 | Medium | 0 matches across `apps` **[V]** |
| G-2 | **Rooming list** (view + import/export) — none | D3/D4 | High (group ops) | 0 matches for `roomingList` in modules **[V]** |
| G-3 | **Structured search parameters ignored** (`confirmationNo`, `guestName`, `roomNumber`, `companyName`, `agentName`, `marketCode`, `sourceCode`, `reservationType`) | A5/A6 | High | DTO declares them (`search-reservation.dto.ts:16-23`); repo reads only free-text `search` **[V]** |
| G-4 | **Room-number search** — not executed | A5/C4 | High | `:143-157` OR-clause excludes room number **[V]** |
| G-5 | **Optimistic locking / version** on reservations — none; `ConcurrencyConflictError` never thrown | C4/E5 | Critical | `reservation-errors.ts:24`; zero throws **[V]** |
| G-6 | **DB-level room-overlap exclusion constraint** — none | C4 | Critical | 0 `EXCLUDE`/`USING gist`/overlap matches in `packages/db/*.sql` **[V]** |
| G-7 | **Server-side balanced-folio check-out gate** — none | C7/E4 | Critical | `billing.service.ts:878-896`; `check-out.handler.ts:525-530` **[V]** |
| G-8 | **Row-level security policy on reservations** — enabled but policy-less | F4 | High | Only one `CREATE POLICY` in `packages/db` (on `tax_codes`) **[V]** |
| G-9 | **Permission decorators** on `walk-in`, `check-out`, `reverse-checkin` — none | C7/C8/C9 | High | `front-office.controller.ts:85-98` **[V]** |
| G-10 | **Durable command journal for reservation commands** — journal exists but is used only by availability assertions | B7/E5 | Medium | `ReservationOperationJournal` wired in `availability-assertion.service.ts` only **[V]** |
| G-11 | **AR / company account object** in cashiering — none (routing is window-only) | E2 | Medium | cashiering handlers **[V]** |
| G-12 | **Idempotency on reservations billing-adapter write path** — none | E5/F5 | High | `prisma-billing.adapter.ts` marker-string dedupe only **[V]** |
| G-13 | **Cross-property reservation transfer** | F6 | Deferred | ADR-072 **[V]** — **NOT A GAP**; explicitly out of scope |

---

## 5. Defects & Enterprise-Risk Register

Defects are **present-but-wrong** behaviors (distinct from gaps).

| ID | Defect | Impact | Evidence | Tag |
|----|--------|--------|----------|-----|
| D-1 | **Double-assignment race**: room-overlap detection is display-only; the assignment write does not consult `assignedToOtherReservation`, and there is no DB exclusion constraint. Two concurrent assignments to the same room can both succeed (last-write-wins). | Overbooked/duplicate-occupied room at the desk; guest displacement; revenue and reputation risk. | `get-check-in-room-options.handler.ts` / `room-assignable.specification.ts` hydrate the flag; assign-room path does not supply it; no `EXCLUDE` constraint **[V]** | [V] |
| D-2 | **Check-out ignores unbalanced folio**: `closeFolio` flips status with no balance check; handler swallows failure. | Unsettled balances leave the property silently; revenue leakage; audit exposure. | `billing.service.ts:878-896`; `check-out.handler.ts:525-530` **[V]** | [V] |
| D-3 | **Walk-in does not check in**: creates `reservation_status:'CONFIRMED'`, no space, no check-in, no room assignment. | Walk-in guests are not actually in-house; contradicts ADR-073 intent and desk workflow. | `sub-resources.handlers.ts:471-490` **[V]** | [V] |
| D-4 | **Unauthenticated-mutation surface**: `walk-in`, `check-out`, `reverse-checkin` routes lack permission decorators while sibling routes (`transfer`, `reinstate`, `extend`, `upgrade`, `change-rate`) require permissions. Controller is `@PropertyScope(false)`. | Privilege escalation; any caller reaching the route can mutate stays. | `front-office.controller.ts:24,85-98` **[V]** | [V] |
| D-5 | **Search contract mismatch**: DTO advertises 8 structured params; only free-text is honored — clients silently get wrong results. | Operator searches by confirmation/room/company silently fail; UX trust erosion. | `search-reservation.dto.ts:16-23` vs `reservation.repository.ts:143-157` **[V]** | [V] |
| D-6 | **RLS enabled but policy-less on reservations**: isolation depends solely on application-level scoping (property-scope guard + ALS repository scope). Any unscoped raw query bypasses isolation. | Multi-tenant data-leak risk if any code path escapes the guard. | `packages/db/*.sql` **[V]** | [V] |
| D-7 | **`closeFolio`/`reopenFolio` optional `hotelId`**: when omitted, the predicate is dropped (`billing.service.ts:878-905`). | Potential cross-property folio mutation if a caller omits `hotelId`. | **[V]** | [V] |
| D-8 | **Group wash scheduler default OFF**: feature flags (`gba.pickup.canonicalRead`, `canonicalWrite`) default OFF. | Allotment release may not run in production posture; unpicked rooms not washed automatically. | `allotment.controller.ts:302`, `reservation-pickup-cascade.service.ts:94` **[V]**; scheduler default **[R]** | [V]/[R] |
| D-9 | **Queue worklists capped ~25 without pager** | Operators cannot page beyond the cap; large arrival days truncate. | **[R]** — not independently re-verified | [R] |
| D-10 | **25 of 76 group-allotment specs are `*.postgres.spec.ts`, skipped without `AVAILABILITY_TEST_DATABASE_URL`** | Many DB-level atomicity proofs never run in default CI. | **[R]** | [R] |

> **Defects D-1..D-8** were confirmed by direct inspection during this audit. **D-9/D-10** come from parallel streams and remain `[R]` until re-verified (§6).

### 5.1 Unresolved items (not asserted)

| ID | Item | Why unresolved |
|----|------|----------------|
| U-1 | Full Mews reservation-state coverage (Optional/Requested/Inquired semantics) in XYLO | Mews article bodies not retrievable; XYLO modeling not exhaustively traced **[U]** |
| U-2 | Mews "bulk group check-in" semantics vs. XYLO `BatchUpdateStatus` | Mews behavior snippet-only **[U]** |
| U-3 | Mews Timeline parity | Snippet-only; XYLO room-grid relationship unclear **[U]** |
| U-4 | Cross-property transfer (Mews) vs ADR-072 deferral | Mews semantics snippet-only **[U]** |
| U-5 | Bulk check-out financial/authorization controls | Partially unverifiable — see §7 Q3 |

---

## 6. Verification Backlog

Ordered by risk-reduction per unit of effort. Each item states **the exact question** and **the minimal test**.

| # | Verify | Method | Closes |
|---|--------|--------|--------|
| VB-1 | Race two concurrent `assignRoom` calls to one room; assert only one commits | Integration test against a disposable DB; assert second returns a conflict | D-1, G-5, G-6 |
| VB-2 | Check out with a non-zero folio balance; assert check-out is blocked (or requires explicit override) | Handler/integration test on `closeFolio` + check-out handler | D-2, G-7 |
| VB-3 | Anonymous/low-privilege principal calls `walk-in`, `check-out`, `reverse-checkin`; assert 403 | Route-level e2e with permission matrix | D-4, G-9 |
| VB-4 | Walk-in end-to-end: assert the reservation reaches `Started`/checked-in with a space | Handler test on `WalkInCommandHandler` | D-3 |
| VB-5 | Search with `roomNumber`, `confirmationNo`, `companyName`; assert correct rows | Repository/query test | D-5, G-3, G-4, ED-1 |
| VB-6 | Confirm reservations RLS posture in a live DB: `pg_policies` for `reservations` empty | Read-only catalog query on a disposable DB (never `xylo_cloud`) | D-6, G-8 |
| VB-7 | Call `closeFolio` with omitted `hotelId` from hotel A against hotel B's reservation id | Unit/integration | D-7 |
| VB-8 | Deposit → folio → check-out lifecycle: confirm no duplicate postings on retry (reservations adapter path) | Integration with replay | G-12, E5, Q5 |
| VB-9 | Confirm queue worklist cap behavior + absence of pager | Read `get-arrivals/departures` handlers; UI pagination | D-9 |
| VB-10 | Run group-allotment `*.postgres.spec.ts` with `AVAILABILITY_TEST_DATABASE_URL` set | CI job on disposable DB | D-10 |
| VB-11 | Re-confirm Mews behaviors tagged `[U]` against official Mews sources via a browser session (Help Center SPA renders in a real browser) | Manual/session-based retrieval | U-1..U-4, §0.2 |

---

## 7. Answers to the Six Required Questions

**Q1 — Search by reservation/confirmation number.**
*Partially implemented.* Free-text `search` matches `confirmation_no` and reservation `id` (`reservation.repository.ts:152-157`) **[V]**, so confirmation-number and reservation-id lookups work **via free text**. But the DTO's dedicated `confirmationNo` parameter is **ignored**, and **room number is not matched at all**. Classification: **IMPLEMENTED—VERIFICATION INCOMPLETE** with defect **D-5 / G-4**.

**Q2 — Overlapping room assignment under concurrency.**
*Not protected.* There is **no optimistic lock/version** (`ConcurrencyConflictError` defined but never thrown, `reservation-errors.ts:24` **[V]**) and **no DB exclusion constraint** (0 `EXCLUDE` matches **[V]**). Overlap detection is computed for the check-in room-option list but **not enforced at the assignment write** **[V]**. Classification: **IMPLEMENTED BUT DEFECTIVE** (defects **D-1**, gaps **G-5/G-6**). This is the single highest enterprise risk in scope.

**Q3 — Bulk check-out financial/authorization controls.**
*Not enforced and partly unverifiable.* `closeFolio` performs **no balance gate** (`billing.service.ts:878-896` **[V]**) and the check-out route carries **no permission decorator** (`front-office.controller.ts:90-93` **[V]**). Whether a separate "bulk" check-out path exists with different controls is **not confirmed** (no bulk check-out route found), so the *bulk-specific* question is **UNRESOLVED**; the *single* check-out control gap is **CONFIRMED**. Classification: **IMPLEMENTED BUT DEFECTIVE / PARTIALLY UNRESOLVED**.

**Q4 — Saved views / grouping.**
*Missing.* **0 matches** for saved views/filters across `apps` **[V]**. No grouping-persistence was found. Classification: **MISSING** (gap **G-1**).

**Q5 — Deposits / unallocated payments across the reservation+folio lifecycle.**
*Implemented with unequal assurance.* Deposits exist (`reservation_payment`, `PostDeposit`, `TransferDeposit`) and `transferDepositToFolio` applies a `${hotelId}:${reservationId}` marker to avoid re-posting **[V]**. However, the **billing module** provides full idempotency (`idempotencyKey` + `externalReference` + advisory lock + in-transaction re-check: `billing.service.ts:127-165` **[V]**), while the **reservations billing-adapter write path does not** (**G-12**) **[V]**. The deposit-to-folio reconciliation across the full lifecycle is therefore **IMPLEMENTED—VERIFICATION INCOMPLETE** with a material assurance asymmetry. Classification: **PARTIALLY IMPLEMENTED**.

**Q6 — Financial-action traceability and duplicate/missing-posting prevention.**
*Split posture.* Billing module: **strong** — idempotency keys, external references, folio advisory lock, in-transaction replay check, and static gates (`t2-option-b.phase3.spec.ts`, `folio-tax-details-integrity.phase3.spec.ts`) **[V]**. Reservations billing adapter: **weak** — marker-string dedupe only, no idempotency key **[V]**. Durable command journal exists but is **wired only for availability assertions**, not reservation commands (**G-10**) **[V]**. Classification: **PARTIALLY IMPLEMENTED** (gap **G-12**, defect-adjacent risk).

---

## 8. Prioritized Recommendations

**These are recommendations only. No remediation is authorized by this audit.**

### P0 — Enterprise-critical (block parity claim)
1. **Introduce a concurrency guard for room assignment.** Add an optimistic version column (or row lock) on reservations and a DB exclusion/overlap constraint on (room, stay-range) assignments; throw `ConcurrencyConflictError` on conflict. *(D-1, G-5, G-6)* — closes the highest risk in scope.
2. **Enforce a balanced-folio check-out gate** server-side, with an explicit, authorized, audited override matching Mews' "check out with unbalanced bill." *(D-2, G-7)*
3. **Add permission decorators** to `walk-in`, `check-out`, `reverse-checkin` (and audit the remaining unscoped routes on `FrontOfficeController`). *(D-4, G-9)*

### P1 — High
4. **Fix walk-in** to actually check the guest in and assign a space (align with ADR-073). *(D-3)*
5. **Honor the structured search contract** (`confirmationNo`, `guestName`, `roomNumber`, `companyName`, `agentName`, `marketCode`, `sourceCode`, `reservationType`) or narrow the DTO to what is implemented; make room number searchable. *(D-5, G-3, G-4)*
6. **Add RLS policies** for reservations (or document and test the application-level boundary as the deliberate control). *(D-6, G-8)*
7. **Close the idempotency asymmetry** on the reservations billing-adapter write path. *(G-12, Q5/Q6)*

### P2 — Medium
8. **Rooming list** (view + import/export) for group operations. *(G-2, D3/D4)*
9. **Saved views / saved filters.** *(G-1)*
10. **Make `hotelId` mandatory** in `closeFolio`/`reopenFolio`. *(D-7)*
11. **AR/company account object** (or document master-folio-only as the intended model). *(G-11, E2/ED-4)*
12. **Wire the durable command journal** into reservation commands. *(G-10)*
13. **Enable and verify the group wash scheduler** posture for production. *(D-8)*
14. **Re-run the skipped `*.postgres.spec.ts` groups** in CI on a disposable DB. *(D-10)*
15. **Queue worklist pagination** beyond ~25. *(D-9)*

---

## 9. Final Verdict

### **PARTIAL PARITY WITH MATERIAL GAPS**

**Rationale.**

- XYLO **substantially matches** Mews across the operational core: reservation discovery and filtering, arrival/departure worklists, reservation detail, front-desk arrival/departure with room readiness, room assignment, room moves, upgrades, overbooking allowances, groups/blocks/allotments, and reservation↔folio coupling with a group master folio for guest-vs-company separation. Several of these are **verified**.
- XYLO falls short of an enterprise-parity claim in three material areas that Mews documents as first-class controls:
  1. **Concurrency** — no real protection against conflicting room assignments (**D-1**).
  2. **Financial control** — no server-side balanced-bill check-out gate (**D-2**), plus an idempotency asymmetry (**G-12**).
  3. **Authorization** — unprotected high-risk mutation routes (**D-4**).
- These are **confirmed** (direct evidence), not unresolved. Additionally, **search precision** (D-5) and **saved views / rooming lists** (G-1/G-2) are confirmed gaps.
- The verdict is **not** "no parity" — the surface is broad and much is verified — and **not** "parity" — the controls above are essential to enterprise hotel operations.

**Decision rule for the claim:**
> Enterprise parity may be asserted **only after** P0 items (concurrency guard, balanced-folio gate, authorization decorators) are implemented **and** independently verified by the tests in the Verification Backlog (§6).

---

## Appendix A — Evidence Inventory

**Directly verified during this audit [V]:**
- `apps/api/src/modules/reservations/infrastructure/repositories/reservation.repository.ts:130-157` — search clause (guest name/confirmation/id only; room number absent).
- `apps/api/src/modules/reservations/api/dto/search-reservation.dto.ts:16-23` — declared structured params.
- `apps/api/src/modules/reservations/api/controllers/reservations.controller.ts:100,108` — `PUT` + `PATCH`.
- `apps/api/src/modules/reservations/domain/errors/reservation-errors.ts:24` — `ConcurrencyConflictError` (never thrown).
- `apps/api/src/modules/reservations/infrastructure/adapters/prisma-billing.adapter.ts:12-263` — folio/posting schema notes, marker dedupe, optional hotelId paths.
- `apps/api/src/modules/billing/billing.controller.ts:9-68` — T-2 removal of `@PropertyScope(false)`; `@Permissions` present.
- `apps/api/src/modules/billing/billing.service.ts:127-165,878-905` — idempotency + advisory lock; `closeFolio`/`reopenFolio` no balance gate, optional hotelId.
- `apps/api/src/modules/front-office/front-office.controller.ts:24,85-98` — `@PropertyScope(false)`; missing decorators on walk-in/check-out/reverse-checkin.
- `apps/api/src/modules/front-office/application/commands/check-out/check-out.handler.ts:525-530` — closeFolio in swallowed try/catch.
- `apps/api/src/modules/front-office/application/commands/sub-resources/sub-resources.handlers.ts:471-490` — walk-in creates CONFIRMED only.
- `apps/api/src/modules/group-allotment/api/controllers/group-booking.controller.ts:126-162` — master-folio endpoints.
- `apps/api/src/modules/group-allotment/api/controllers/allotment.controller.ts:252,302` — wash endpoint + canonical-read flag.
- `packages/db/*.sql` — single `CREATE POLICY` (on `tax_codes`); many `ENABLE ROW LEVEL SECURITY`; no `EXCLUDE` overlap constraint.
- Absence searches: saved views/filters (0 matches); `roomingList` (0 matches).

**Reported by parallel audit streams [R]** (pending §6 re-verification):
- Worklist caps/pager behavior; check-in eligibility/readiness flow; ADR-078 convergence of 7 FO reservation writers; group wash scheduler default OFF; `*.postgres.spec.ts` skip posture.

**Mews sources [M]/[U]:** Mews Help Center article metadata/snippets (Reservations states/origins/detail, Check-in, Check-out, Timeline, Search, deposits, rooming lists, group limits, overbooking, company billing); Mews product page (rate management, group allocations, live availability, block release dates); `docs.mews.com` Connector API (`Get all reservations`, per-property tokens). Article bodies were **not machine-retrievable** (§0.2).

---

*End of audit. No implementation performed. Awaiting approval before any remediation.*

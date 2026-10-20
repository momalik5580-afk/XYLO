# XYLO HOUSEKEEPING — PHASE 1 EXIT ASSESSMENT (DELIVERABLE 6)

Document: 06_PHASE1_EXIT_ASSESSMENT.md
Lifecycle step: Phase 1 — Forensic Audit (gate document)
Status: COMPLETE — DECISION RECORDED BELOW
Date: 2026-10-20

---

## 1. Purpose and decision rule

This document decides whether Phase 1 (Forensic Audit) met its exit criteria before any Phase 2 (Business Rules Decision) work begins. Per 00_AUDIT_CONTROL.md, the mandated rule is:

- **PASS** = working tree state fully surveyed read-only; findings register evidence complete (every finding labeled + classified + line-referenced); all six deliverables present and internally consistent; no business-rule or design decisions smuggled into Phase 1; open questions formally carried forward.
- The known limitation (no runtime execution) is **not a Phase 1 criterion** — Phase 1's charter is repository-state establishment — but it must be explicitly recorded, not silently absorbed. 00 §5 and 05 §4 record it.

## 2. Coverage of the audit charter (mandate item by item)

### 2.1 Covered completely (evidence-complete)

| Charter area | Where recorded |
|---|---|
| `apps/api/src/modules/housekeeping` controller/service/CQRS/events full source read | 02 register (HK-001..HK-037), 03 §1–§9, 04 |
| Frontend: page route, all 12 `_components`, 1,523-line legacy store, React Query feature | HK-001/002/003/022/023/028/029/030, 04 |
| Persistence: schema models (`housekeeping_tasks` 5759–5779, `hk_room_assignments` 5781–5794, `room_status` 10959–10971, `rooms` 11174–11189), HK migrations, ratified baseline declarations | 04, HK-003/HK-012, 00 §6 |
| Cross-module seams actually touched by HK code (FO due-outs/discrepancy, room-service requests, lost-found, employees, command-center widgets) | 03 §3/§4/§7, Annex A, 04 |
| Room-condition writer census + readiness-gate seam (`room.aggregate.ts:30`, `room-assignment.service.ts:55`, checkout write `:470–481`) | 05 §7, HK-006/HK-013–HK-016/HK-019/HK-020 |
| Authorization surfaces (controller 62 routes × no role gates; inspector identity persistence; permission-registry reference) | HK-007/HK-008 |
| Test/runtime posture (0 module tests, 16 peripheral tests, no execution) | 05 |
| Prior audit lineage respected (R8 ledger, baseline declarations, phase-1 closure, OPERA gap analysis) | 00 §6 |

### 2.2 Covered partially (recorded deliberately, not omitted)

- **Supply-request service**: transition map + submit/approve/reject fully verified (HK-037); fulfill/close/partial-fulfillment chains read at contract level only; item-quantity arithmetic (`remaining_quantity = $1 - delivered_quantity`, `:393`) not audited for edge cases.
- **Lost-found & room-service seams**: audited at the contract/cross-walk level (Annex A) — no dedicated defects raised for those modules themselves (out of scope per 00 §2).
- **Employees/HR reference data**: staff-join mismatch verified (HK-012); provisioning write-path absence verified (HK-024); the wider HR module itself unpaid.
- **Command-center HK widget provider**: source read and cross-referenced; not load-bearing for any D/R finding.
- **Packages/cli seeds** for HK tables: read for par/linen seed evidence (HK-035); untracked for other tables.

### 2.3 Not investigated (and why this is acceptable at Phase 1)

- Runtime/DB behavior of any endpoint (no docker/compose/network execution allowed for this audit).
- `apps/mobile` HK surfaces (no HK-specific mobile route found in discovery; if one exists, it is dead code for a dead backend slice — flagged as Phase-2 cleanup item, not a defect).
- Inventory module's own `/supply-requests` stack (used by `apps/web/lib/inventory/...`, distinct from the HK orphan stack of HK-037/Annex A).
- Performance, load, concurrency (Phase 12 concerns per repo roadmap precedent).

## 3. Read-only compliance

- No production, test, schema, migration, config, dependency, or pre-existing documentation file was modified. Only files created/edited in this session: `docs/audit/Housekeeping/00/01/02/03/04/05/06` document set (02 edited to append HK-037 during Deliverable 5 — the register is a Phase-1 deliverable, per the mandate's "only permitted outputs are new audit documents"). VERIFIED by write-set cross-check: every path touched in this audit resolves inside `docs/audit/Housekeeping/`.

## 4. Open questions carried formally into Phase 2

(from register §E/[Q] + 03/04 unanswered items — restated for the gate)

1. Is inspection-approve the *intended* policy for rooms re-entering service, or a legacy accident (HK-006)? Who owns CLEAN/OOO/OOS lifecycles (HK-014/HK-020)?
2. `front_office_status` — dead column or unimplemented owner (HK-015)?
3. Business cadence: daily-generation semantics, business_date, shifts, workload credits, capacity (HK-036/HK-031/HK-004/HK-009) — all decision-ready but undecided.
4. Discrepancy doctrine: whose person-count authority; resurrect `person_discrepancies` table or retire the stack (HK-003 + ratified baseline defect)?
5. Endpoint purge-or-build set for the ~40 dead HTTPS calls (HK-001) and the ~6 dead components (HK-023) — deletion is already pre-approved for backend dead files, not for these.
6. Supply-request intended flow (auto-advance to PENDING_APPROVAL? multi-approver model at `:383–388`?) before any consumer is built (HK-037).
7. Assignment data model: serialize attendants into `rooms` JSON vs the unused `hk_room_assignments`/supervisor-table path (HK-011).
8. Whether the legacy `room_status` mirror is retired or kept as write-through (HK-013).

## 5. Exit-gate evaluation

| # | Criterion (mandate) | Status | Evidence |
|---|---|---|---|
| 1 | Repository state surveyed read-only | MET | 00 §1; §3 above |
| 2 | Scope boundaries established and honored | MET | 00 §2; §2 above (partial/not-investigated explicitly listed) |
| 3 | Findings register complete (labels, classification, file:line) | MET | 02 (HK-001..HK-037; Annex A/B) |
| 4 | Six deliverables present and consistent | MET | 00 §5 index ↔ actual files; cross-references resolve |
| 5 | Cross-module seams documented | MET | 03 §3/4/7; Annex A; 05 §7 |
| 6 | No business/design decisions made in Phase 1 | MET | §E items labeled "not proposals"; §4 items are questions, not decisions |
| 7 | Runtime limitation recorded | MET | 00 §4/§5; 05 §4/§6 |
| 8 | Prior-audit lineage preserved and cited, not overwritten | MET | 00 §6 |

## 6. Decision

**PASS — Phase 1 exit gate is met**, with one recorded caveat, not a criterion failure:

- **Caveat (runtime)**: All execution-class claims (HTTP behavior, DB round-trips, UI rendering) are OBSERVED BEHAVIOR per 05 §4. HK-003's missing-table verdict rests on the ratified `migration-baseline.declarations.json:89–99` rather than a live negative probe. This is acceptable per the Phase 1 charter (establish verified current state from repository evidence), and every affected claim is labeled.

**Phase 2 entry preconditions (for the next phase's own gate, not this one):**

1. Run the runtime verification list (05 §4/§7: live check `HK-012` join against `employees.id` FK; supply-request transition behavior vs HK-037; `hk_sla_config` write-failure tolerance; discrepancy endpoint 404 + payload mismatch probe) to convert the remaining OBSERVED items to VERIFIED where the decision depends on them.
2. Confirm the audit doc set is reviewed/accepted by the module owner before Business Rules Decision workshops use it as the evidence base.

## 7. Recommended next step (not started)

Phase 2 — **Business Rules Decision**: a doc-driven decision cycle over register items that are `Q`/`R`-shaped and of the six load-bearing question sets in §4 above, producing the business-rule ledger the final domain specification (Phase 3) will be written against. This recommendation is **not initiated by this audit** per the mandate's no-Phase-2-advance clause.

## 8. Sign-off block

| Role | Name | Position | Date |
|---|---|---|---|
| Author (forensic audit session) | AI agent (read-only investigation) | PASS recorded above | 2026-10-20 |
| Module owner | — (pending review, precondition §6.2) | — | — |

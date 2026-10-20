# XYLO HOUSEKEEPING — PHASE 3 TRACEABILITY

Document: 02_TRACEABILITY.md
Lifecycle step: Phase 3 — Final Domain Specification
Status: COMPLETE — 32/32 HKD records and 37/37 Phase 1 findings mapped; zero "unaddressed" rows
Date: 2026-10-20

Labels are the verified classes of `00 §4` (source: Phase 2 `02 §0/§1`). FDS references use rule IDs `F<c>.<n>` from `01_FDS_SYNTHESIS.md`. Dispositions: **specified** (rule fixed in spec), **open-input** (ODI reference), **out-of-scope** (recorded non-goal with reopen condition), **external-gated** (unchanged normative text; activation or execution requires a named human/external act).

## Part A — Decision records (32/32)

| HKD | Label | FDS rules | Disposition |
|---|---|---|---|
| HKD-01 | EB | F2.1, F2.4, F2.7, F13.2 | specified (constants alignment = implementation-stage) |
| HKD-02 | APPROVED | F2.3, F2.8, F4.8, F5.1, F5.3, F14.4 | specified (anchor verbatim; access-control half external-gated via OQ-12) |
| HKD-03 | EB | F5.2, F5.4, F5.5, F5.8, F4.5 | specified (auto re-clean = requirement; execution later) |
| HKD-04 | DEL | F9.1, F9.2, F9.3, F9.4 | specified; placement roles = ODI-05; external-gated (bundle) |
| HKD-05 | EB | F2.2, F2.8, F2.9 | specified |
| HKD-06 | DEL | F2.6, F7.2, F13.1, F13.2 | specified; same-wave constraint; FO co-sign carried; external-gated (bundle) |
| HKD-07 | EB | F2.10, F7.5 | specified (excluded from HK domain) |
| HKD-08 | EB | F2.10, F7.5 | specified (excluded from HK domain) |
| HKD-09 | APPROVED | F3.1, F3.6, F3.8 | specified (anchor verbatim) |
| HKD-10 | PROV | F1.1 | out-of-scope (reopen condition recorded) |
| HKD-11 | EB | F3.3, F3.4 | specified; key composition = ODI-01 |
| HKD-12 | DEF | F1.3, F10.3 | out-of-scope (accepted deferral, reconsideration condition recorded) |
| HKD-13 | APPROVED (derived) | F3.2 | specified (anchor verbatim) |
| HKD-14 | EB | F3.5 | specified; naming/merge = ODI-02 |
| HKD-15 | EB | F4.6 | specified (reopen condition documented) |
| HKD-16 | APPROVED | F8.1, F8.4, F13.3 | specified (stack-retirement execution in later gates) |
| HKD-17 | APPROVED (derived) | F8.2, F7.2 | specified |
| HKD-18 | EB | F8.3 | specified |
| HKD-19 | APPROVED | F1.7, F1.9, F1.10 | specified (funnel operational) |
| HKD-20 | DEL | F4.7, F10.1, F10.2, F13.3 | specified; persistence = later implementation; external-gated (bundle) |
| HKD-21 | DEL | F11.1, F11.2 | specified; ops inputs ODI-06/09; external-gated (bundle) |
| HKD-22 | PROV | F11.3, F1.2 | out-of-scope (reopen condition recorded) |
| HKD-23 | DEL | F6.1, F6.2, F6.5, F1.5, F13.3 | specified; **correction #4 resolution recorded in `00 §3`** (teams-retirement ≠ provisioning scope); external-gated (bundle) |
| HKD-24 | EB | F4.8, F12.4 | specified (later-phase integration) |
| HKD-25 | EB | F1.9, F12.4, F13.3, Ch 15 row | specified (UI-phase disposition) |
| HKD-26 | EB | F1.9, F12.4, F13.3 | specified (retirement list) |
| HKD-27 | EB | F12.1, F12.2, F4.4, F11.2, F13.1 | specified; endpoint adoption-or-removal = ODI-08 |
| HKD-28 | APPROVED | F1.1, F11.4, Ch 15 supplies row | specified (retirement execution in later gates) |
| HKD-29 | DEF | F11.4 | deferred (inventory-domain obligation, recorded) |
| HKD-30 | DEL | F4.1, F6.3, F6.4 | specified; shape = ODI-03; external-gated (bundle) |
| HKD-31 | EB | F2.5, F2.8, F5.6, F9.2, F8.2-adjacent | specified |
| HKD-32 | RETAIN | F2.10, F7.1, F7.3, F7.4 | specified (retention); removal = external-gated (later DB-phase gate, 4 prerequisites) |

Coverage: **32/32 specified / open-input / out-of-scope / external-gated — zero unaddressed.**

## Part B — Phase 1 findings (37/37)

| HK | Class | Spec resolution | FDS anchor | Later-phase owner (for deferred/retired executions) |
|---|---|---|---|---|
| HK-001 | D | Specified as retirement/integration lists; dead URLs prohibited | F1.9/F1.10, F13.3, Ch 10/11/Ch 6 move-room | UI phases (removals); later phases (integrations) |
| HK-002 | D | Prohibited (fabricated movements) | F11.2, F13.3 | Linen implementation phases |
| HK-003 | D | Stack retired (dead in spec); FO stack sole authority | F8.1, F8.4 | Stack retirement execution |
| HK-004 | D | Resolved by approved ensure-missing + invariants | F3.1, F3.2, F3.8 | Generation implementation |
| HK-005 | D | Resolved in spec: system-attributed collision-safe ids; server allocates | F3.3, F3.4, F4.5, F15.1 | Generation/identity implementation |
| HK-006 | D | Resolved: approved-only world + CLEAN producer rule | F4.3, F5.1, F5.2 | Task-completion implementation |
| HK-007 | D | Recognized; resolution is the authorization matrix | F14.1–F14.5 | **OQ-12 sign-off → implementation** |
| HK-008 | D | Resolved: authenticated identity; client fields prohibited | F4.5, F5.2 | Inspection implementation |
| HK-009 | D | Dies in spec: system ids + no mid-run abort | F3.3, F3.6 | Generation implementation |
| HK-010 | D | Boundary note: mutation-verified counts required if/when formalized in the owning domain | F8.4-adjacent, Ch 15 FO row | FO domain (retired HK stack) |
| HK-011 | D | Resolved: task-based ownership; card/plan split; reassign writes supervisor ids on cards only | F4.1, F6.3, F6.4 | Assignments implementation |
| HK-012 | D | Resolved: FK-backed join rule | F6.6, F6.5 | Staff provisioning implementation |
| HK-013 | R | Resolved: retention contract; no new consumers | F7.1, F7.3, F2.10 | Mirror contract until later DB gate |
| HK-014 | R | Resolved: CLEAN produced by cleaning-task completion | F2.3, F4.3 | Task-completion implementation |
| HK-015 | R | Resolved: FO-owned re-scope (same wave); column demoted | F2.6, F7.2, F13.1 | FO domain (co-sign) + implementation order |
| HK-016 | R | Resolved: legacy gate aligned to canonical gate | F2.8 | Check-in pipeline implementation (FO/HK) |
| HK-017 | R | Boundary rule: no runtime DDL; tenant-true scoping | F15.1 | Service-layer implementation |
| HK-018 | R | Resolved: fail-loud config writes; no success-without-persistence | F4.4 | Config implementation |
| HK-019 | R | Boundary: literals normalize per maintenance contract; FO value-object = FO domain | F9.4 | Maintenance domain + FO boundary |
| HK-020 | R | Resolved: maintenance owns lifecycle; HK consumes | F9.1, F9.2 | Maintenance domain (implementation) |
| HK-021 | R | Resolved: one vocabulary per axis/casing-alignment implementation | F2.1, F2.4, F2.7 | Constants alignment |
| HK-022 | G | Prohibited: server-authoritative counters; fabrications specified out | F12.1, F12.2 | Dashboard implementation |
| HK-023 | G | Retirement list (incl. crash-if-render component) | F1.9, F12.4, F13.3 | UI phases |
| HK-024 | R | Resolved: provisioning write path = spec requirement | F6.5, F6.6 | Staffing implementation |
| HK-025 | G | Explicit not-a-Phase-2-rule disposition: no mobile surface specified | F13.4, F8.4 | None (owner product decision would be a new act) |
| HK-026 | I | Resolved: enforced transition matrix | F2.7, F4.2 | Lifecycle enforcement implementation |
| HK-027 | I | Resolved: canonical gate + documented consumer tolerances | F2.8, ODI-12 | Consumer alignment implementation |
| HK-028 | I | Prohibited: no fabricated conditions | F13.1, F2.9 | UI implementation |
| HK-029 | I | Resolved: axis separation | F13.2 | UI + FO-owned defect sites |
| HK-030 | I | Resolved: string protocol retired; no fire-and-forget completion | F4.7, F10.2 | Request-contract implementation |
| HK-031 | I | Out-of-scope: no shift concept (single daily business-date model) | F1.1, F3.3 | Reopen on HKD-10 evidence |
| HK-032 | E | Flagged: consolidation decision (reassign vs send-back) | ODI-11, F4.6 | Engineering |
| HK-033 | E | UI-phase refresh rule (not-a-Phase-2-rule origin) | F12.3 | UI phases |
| HK-034 | E | Flagged: endpoint adoption-or-removal | ODI-08, F12.1 | Review per HKD-27 |
| HK-035 | E | Resolved: real model required; synthesis prohibited | F11.2, ODI-06 | Linen implementation |
| HK-036 | Q | Resolved across the register: date key (F3.3), shifts excluded (F1.1), credits absent (F1.3), trigger manual-only (F3.6) | Ch 1/3 | Per ODI owners |
| HK-037 | D | Resolved: stack retired; lifecycle handed to inventory domain | F11.4, Ch 15 inventory row | Inventory domain |

Coverage: **37/37 with verdicts — zero unaddressed.** Deferred-to-implementation rows all name their owning chapter; retired-stack rows name their execution gates (`04_EXTERNAL_GATES_AND_CONDITIONS.md`).

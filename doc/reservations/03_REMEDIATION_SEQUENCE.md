# Reservations — Proposed Remediation Sequence

**Date:** 2026-10-16
**Companion:** `01_CAPABILITY_AUDIT.md`, `02_GAP_REGISTER.md`
**Status:** PROPOSAL ONLY. This document does not authorize execution. Ordering follows booking integrity → security → revenue-critical → operational quality. It does not invent new phases; it sequences the gaps already registered in `02_GAP_REGISTER.md`.

Ordering principles:
1. **Integrity and security before surface features** — never ship a widening change on top of a known integrity or auth hole.
2. **Unblock revenue intake before polishing reporting.**
3. **Resolve ownership boundaries before touching the code inside them** (deposits).
4. **Prove integrity with executable tests, not documentation.**

---

## Wave 0 — Safety net (do first, small, low-risk)

Purpose: make the integrity claims executable so later waves cannot silently regress them.

| Gap | Action | Why first |
|-----|--------|-----------|
| RG-13 | Provide `AVAILABILITY_TEST_DATABASE_URL` (CI job or documented local harness) and run the availability concurrency/lock-order/isolation Postgres specs; record results. | Every later change to lifecycle/availability must be guarded by these. |
| RG-12 | Collapse to a single `PermissionGuard` `APP_GUARD`. | Prevents confusing authz behavior while RG-07 is fixed. |

Exit gate: the availability integrity suites execute and pass in a defined environment; one permission guard is registered.

---

## Wave 1 — Booking integrity & authorization (highest risk if left)

| Gap | Action | Dependency |
|-----|--------|-----------|
| RG-01 | Make the webhook surface explicitly public at the guard layer (e.g. `@Public()` on `WebhookController` + a guard-level exemption for property/tenant scope), so signed OTA callbacks reach `WebhookService`. Add a route-level test through the real guard chain. | Wave 0 (integrity net) |
| RG-08 | Thread a channel-level idempotency key from the webhook into the availability assertion key so duplicate OTA deliveries cannot double-book, including concurrently. | RG-01 |
| RG-07 | Add explicit `@Permission` codes to `ChannelsController` (and confirm handler-level authorization on channel mutation); ensure `PermissionGuard` remains fail-closed for admin surfaces that declare permissions. | Wave 0 (RG-12) |
| RG-11 | Remove the `getHotelId() || params.hotelId` fallback in `reservation.repository.ts` `findAll`; derive scope solely from the request context. | — |

Exit gate: an unauthenticated, validly-signed OTA webhook creates a hotel-scoped reservation and a replay creates none; channel admin routes require an explicit permission; reservation search scope derives only from the resolved context.

---

## Wave 2 — Revenue intake (unblock channels, site booking decision)

| Gap | Action | Dependency |
|-----|--------|-----------|
| RG-02 | Either implement a real outbound distribution adapter with per-channel results, or explicitly de-scope the feature and stop emitting fabricated metrics (`revenue`, `lastSyncStatus`) and `'queued'` success semantics. | RG-01 (inbound contract stable) |
| RG-09 | Decide: implement a public/rate-limited direct booking intake, or record an explicit `OUT OF SCOPE` decision. | — |

Exit gate: channel operations either perform real distribution with honest status, or the UI/product clearly communicate non-implementation; direct booking has an explicit disposition.

---

## Wave 3 — Deposit/guarantee ownership (financial integrity)

| Gap | Action | Dependency |
|-----|--------|-----------|
| RG-06 | Establish a single authoritative deposit ledger and a single shared port. Target: **Billing owns the ledger; Reservations owns policy + HTTP surface.** Migrate the Reservations writer/reader and the billing adapter onto that one port; eliminate the `folio_payments` vs `reservation_deposits` split. | Wave 1 (authz) |
| RG-14 | Wire `DepositPolicyService` into the create/guarantee paths (or remove it and document the real rule). | RG-06 (agreed ownership) |

Exit gate: a deposit posted through the Reservations API is visible to the billing/settlement path and vice-versa; deposit requirement is enforced by a single rule with a test.

> Note: RG-06 is a design/ownership decision that should be ratified before code changes, consistent with the existing ADR practice.

---

## Wave 4 — Search, saved views, and history (operational quality)

| Gap | Action | Dependency |
|-----|--------|-----------|
| RG-03 | Make every declared search filter actually narrow results (with tests), or remove the dead filters from the DTO and UI. | — |
| RG-04 | Persist display sets (create a real store) or return documented defaults without querying a missing table; make the Sales Screen selector work. | — |
| RG-05 | Add a hotel-scoped read endpoint over `reservation_status_history` / `reservation_changes`; replace the fabricated `AuditPanel` timeline with real data. | — |
| RG-10 | Implement rooming-list import/export, or remove the non-functional buttons with a documented decision. | — |

Exit gate: field filters narrow results; saved views load without error; the History tab shows authoritative rows; rooming-list bulk actions either work or are removed.

---

## Dependency Graph (compact)

```
Wave 0 (RG-13, RG-12)
   └─▶ Wave 1 (RG-01 → RG-08; RG-07; RG-11)
          ├─▶ Wave 2 (RG-02, RG-09)
          └─▶ Wave 3 (RG-06 → RG-14)
Wave 4 (RG-03, RG-04, RG-05, RG-10) — independent of Waves 1–3, gated by Wave 0 only
```

## Explicitly out of scope for this remediation proposal

- Capabilities 8 and 9 (Front Office check-in/out, room readiness/departure worklists) — not Reservations gaps.
- `TransferReservation` cross-property transfer — remains deferred per ADR-072.
- The pre-existing red frontend test `features/reservations/__tests__/b31-fnd09.test.ts` — unrelated to these gaps; note only.

## Approval note

No work in any wave is authorized by this document. Each wave requires review and approval, and Wave 3 additionally requires an ownership ratification before implementation.

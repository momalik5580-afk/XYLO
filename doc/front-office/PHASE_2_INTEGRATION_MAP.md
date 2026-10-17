# XYLO Front Office — Phase 2: Reservation ↔ Front Office Integration Map

- **Date:** 2026-10-17
- **Phase:** Phase 2 — Reservations ↔ Front Office Integration Audit (audit-only)
- **Companion documents:** `docs/front-office/PHASE_2_RESERVATIONS_INTEGRATION_AUDIT.md`, `docs/front-office/PHASE_2_INTEGRATION_GAP_REGISTER.md`, `docs/front-office/PHASE_2_BASELINE_AND_EXIT_GATE.md`

## Legend

| Style | Meaning |
|---|---|
| Green | Verified / working integration seam (direct source and/or test) |
| Red | Confirmed integration defect (`FO-P2-*`) |
| Amber | Suspected / partially verified / not runtime-executed |
| Grey | External module or data store (not fully traced here) |

All diagrams describe the **working tree** at `HEAD = ed278bc`. They are descriptive, not prescriptive.

---

## 1. Overview — Reservations ↔ Front Office Seams

```mermaid
flowchart LR
  subgraph RES["Reservations module"]
    RUI["Reservations UI<br/>ReservationsHomeView / DetailWorkspace"]
    RACT["Action engine<br/>reservation-actions.ts"]
    RAPI["ReservationsController<br/>/reservations/*"]
    RSVC["ReservationsService"]
    RCMD["Owner commands<br/>ExtendStay / Reinstate / NoShow / Cancel"]
    RREPO["IReservationRepository<br/>single-writer methods"]
  end

  subgraph FO["Front Office module"]
    URL["FO URL sync<br/>tab/dialog/reservation"]
    HOST["OperationDialogHost"]
    FOUI["FO dialogs + rack + arrivals"]
    FAPI["FrontOfficeController<br/>/front-office/*"]
    CRS["CrsFrontDeskIntegrationService<br/>(facade + CRS restriction check)"]
    FOSVC["FrontOfficeService<br/>thin orchestration"]
    FOCHK["check-in controller<br/>/check-in/*"]
    FOCMD["FO commands<br/>checkout / reverse / transfer / upgrade / change-rate"]
    FOQRY["FO queries<br/>dashboard / grid / arrivals / due-outs"]
  end

  RUI --> RACT
  RACT -->|"buildCheckInHref(UUID)"| URL
  URL --> HOST --> FOUI
  FOUI --> FAPI
  FAPI --> CRS --> FOSVC
  FOSVC -->|"ExtendStay / Reinstate"| RCMD
  FOSVC --> RREPO
  FOSVC --> FOCMD
  FOCMD -->|"applyCheckOut / applyCheckIn etc."| RREPO
  RAPI --> RSVC --> RCMD
  RCMD --> RREPO
  FOUI --> FOCHK
  FOCHK --> FOCMD
  FOUI -->|"add */extend *"| RAPI
  RSVC -->|"create reservation"| DB[("reservations / rooms")]
  RREPO --> DB
  FOCMD --> DB
  FOQRY --> DB

  classDef confirmed fill:#d1fae5,stroke:#059669,color:#065f46
  classDef defect fill:#fee2e2,stroke:#dc2626,color:#991b1b
  classDef unverified fill:#fef3c7,stroke:#d97706,color:#92400e
  classDef external fill:#e5e7eb,stroke:#6b7280,color:#374151

  class RACT,URL,HOST confirmed
  class RREPO,RSVC confirmed
  class DB external
```

**Notes**
- The facade `CrsFrontDeskIntegrationService` (`crs-front-desk-integration.service.ts:14-20`) sits between the controller and `FrontOfficeService`; it is pass-through except for a CRS restriction check on `changeRateCode` (`:97-115`).
- `FO-P2-002` and `FO-P2-009` are the two seams where the UI reaches the Reservations API with the wrong contract/vocabulary.

---

## 2. Check-in Hand-off and Staged Flow (confirmed, with one defect branch)

```mermaid
sequenceDiagram
  autonumber
  participant RU as Reservations UI
  participant AE as Action engine
  participant URL as FO URL sync
  participant HOST as OperationDialogHost
  participant D as CheckInDialog
  participant G as CheckInGuestList
  participant CI as /check-in API
  participant R as /reservations API
  participant DB as reservations / reservation_guests

  RU->>AE: openCheckIn (confirmed/guaranteed, arrival day)
  AE->>URL: /front-office?tab=arrivals&dialog=check-in&reservation=<UUID>
  URL->>HOST: dialog=check-in + reservation=<UUID>
  HOST->>D: reservationId=<UUID>
  D->>CI: eligibility / flow / rooms / assign / verify / keys / finalize (rid = UUID)
  CI->>DB: findFirst reservations WHERE id=<UUID>
  Note over D,G: FO-P2-001 — same dialog passes confirmationNo to the guest list
  D->>G: reservationId = reservation.confirmationNo
  G->>R: GET /reservations/<confirmationNo>/parties
  R->>DB: SELECT reservation_guests WHERE reservation_id=<confirmationNo>
  DB-->>G: [] (no rows) — silent empty section
```

**Confirmed:** the staged-flow calls use the UUID (`CheckInDialog.tsx:205,252,266,285,298,312`; `check-in-workflow.service.ts:39-46`).
**Defect:** the guest/shares panel uses `confirmationNo` (`CheckInDialog.tsx:556`; `CheckInGuestList.tsx:73`; `reservation.repository.ts:1458-1463`).

```mermaid
flowchart TD
  A["Arrival / rack row<br/>(UUID)"] --> B["CheckInDialog rid = UUID"]
  B --> C["check-in eligibility<br/>loadReservation(UUID)"]
  B --> D["assign room<br/>AssignRoomCommand"]
  D --> E{"RoomAssignableSpec<br/>assignedToOtherReservation?"}
  E -->|"never hydrated"| F["overlap branch unreachable<br/>FO-P2-007"]
  D --> G["room lock + room status transition<br/>(effective safeguard)"]
  B --> H["guest/shares panel<br/>confirmationNo"]
  H --> I["parties query on UUID column<br/>empty result FO-P2-001"]

  classDef defect fill:#fee2e2,stroke:#dc2626,color:#991b1b
  classDef unverified fill:#fef3c7,stroke:#d97706,color:#92400e
  classDef confirmed fill:#d1fae5,stroke:#059669,color:#065f46
  class F,I defect
  class E unverified
  class C,D,G confirmed
```

---

## 3. Lifecycle Ownership — Who Writes What

```mermaid
flowchart LR
  subgraph FOENTRY["FO entry"]
    CO["check-out"]
    RC["reverse check-in"]
    TR["transfer"]
    UP["upgrade"]
    CR["change-rate"]
    EX["extend"]
    RI["reinstate"]
    NS["mark no-show"]
  end

  subgraph OWNER["Authoritative owner"]
    FOOWN["FO-owned handlers"]
    RESOWN["Reservations owner commands"]
    REPO["IReservationRepository<br/>applyCheckOut / applyReverseCheckIn /<br/>assignRoomNumber / applyUpgrade /<br/>applyRateCodeChange"]
    PROC["ProcessNoShowCommand /<br/>CancelReservationCommand"]
  end

  CO --> FOOWN
  RC --> FOOWN
  TR --> FOOWN
  UP --> FOOWN
  CR --> FOOWN
  EX --> RESOWN
  RI --> RESOWN
  NS -.->|"broken: generic PUT"| PROC

  FOOWN --> REPO
  RESOWN --> REPO
  RI -->|"post-commit"| REPO
  PROC --> REPO

  classDef confirmed fill:#d1fae5,stroke:#059669,color:#065f46
  classDef defect fill:#fee2e2,stroke:#dc2626,color:#991b1b
  classDef unverified fill:#fef3c7,stroke:#d97706,color:#92400e

  class CO,RC,TR,UP,CR,EX,RI,FOOWN,RESOWN,REPO confirmed
  class NS,PROC defect
```

**Verified by test (direct + test-derived):**
- `ExtendStayCommand` payload mapping and `ReinstateReservationCommand` dispatch, with reinstate room write through `assignRoomNumber` and *no* `prisma.reservations.update` — `apps/api/src/modules/front-office/__tests__/command-ownership.spec.ts` (PASS, 8 tests).
- `@Permission` enforcement on reinstate/transfer/extend/upgrade/change-rate, including property-scope `REFUSED` — `fo-route-permissions.spec.ts` (PASS, 17 tests).

**Defect:** no-show does not use `ProcessNoShowCommand` (`FO-P2-002`).

---

## 4. Check-out — Financial Side Effects vs. Status Transition

```mermaid
flowchart TD
  U["CheckoutFlowModal<br/>button enabled if paymentMethod"] -->|"POST /front-office/check-out<br/>NO Idempotency-Key"| H["CheckOutCommandHandler"]
  H --> A1["postCredit: early-departure refund"]
  A1 --> A2["postCharge: penalty"]
  A2 --> A3["postCharge: minibar"]
  A3 --> A4["postPayment"]
  A4 --> A5["postCredit + voucher balance/redemption"]
  A5 --> G{"assertTransition(from, CHECKED_OUT)"}
  G -->|"illegal on duplicate"| X["throws — but side effects already committed"]
  G -->|"ok"| T["transaction:<br/>applyCheckOut + status history +<br/>master-folio auto-post"]
  T --> C["closeFolio (legacy family)<br/>errors swallowed"]
  C --> DONE["CHECKED_OUT"]

  classDef defect fill:#fee2e2,stroke:#dc2626,color:#991b1b
  classDef unverified fill:#fef3c7,stroke:#d97706,color:#92400e
  classDef confirmed fill:#d1fae5,stroke:#059669,color:#065f46
  class A1,A2,A3,A4,A5,X defect
  class C unverified
  class T,DONE confirmed
```

**Confirmed defect:** the pre-transition side effects are outside the idempotency/transition protections (`FO-P2-003`). The status write itself is protected (`CHECKED_OUT → CHECKED_OUT` is illegal — `reservation-state-machine.ts:39`).

**Suspected / policy-dependent:** no server-side settlement assertion before `closeFolio` (`FO-P2-008`).

```mermaid
flowchart LR
  subgraph LEGACY["Legacy folio family"]
    FH["folio_header<br/>folio_id Int PK"]
    FC["folio_charges<br/>folio_id = Int header id"]
    FP["folio_payments<br/>folio_id = reservation UUID"]
    FT["folio_tax_details"]
  end
  subgraph NEW["Posting ledger"]
    FS["folios (raw SQL only)"]
    FPOST["folio_postings"]
  end
  BE["BillingService<br/>postCharge / postCredit / postPayment / closeFolio"]
  CE["Cashiering / group-allotment /<br/>check-out master-folio auto-post"]
  UI["CheckoutFlowModal<br/>merges both client-side"]

  BE --> FH
  BE --> FC
  BE --> FP
  BE --> FT
  CE --> FS
  CE --> FPOST
  UI -->|"/billing/folio/:id"| FH
  UI -->|"cashiering useReservationFolios"| FS

  classDef unverified fill:#fef3c7,stroke:#d97706,color:#92400e
  classDef external fill:#e5e7eb,stroke:#6b7280,color:#374151
  class UI,FP unverified
  class FS,FPOST external
```

**Potential risk (`FO-P2-006`):** keys collide across families and within the legacy family (`folio_header.folio_id` Int vs `folio_payments.folio_id` UUID); no server-authoritative merged balance was found.

---

## 5. Authorization Enforcement

```mermaid
flowchart TD
  REQ["HTTP request"] --> JWT["JwtAuthGuard<br/>(authentication)"]
  JWT --> PS["PropertyScopeGuard<br/>(short-circuits on @PropertyScope(false))"]
  PS --> MT["MultiTenantGuard"]
  MT --> ROLE["RolesGuard<br/>(no-op without @Roles)"]
  ROLE --> COMMON{"Common @Permissions<br/>metadata present?"}
  COMMON -->|"yes"| CP["common PermissionGuard<br/>reads request.user.permissions"]
  CP -->|"never populated"| ADM{"role ADMIN/SUPER_ADMIN?"}
  ADM -->|"yes"| OK1["allow"]
  ADM -->|"no"| DENY1["403 — non-admins locked out<br/>FO-P2-005"]
  COMMON -->|"no"| PLAT{"Platform @Permission<br/>metadata present?"}
  PLAT -->|"yes"| RES["PermissionResolverService<br/>hasAllPermissions(userId, perms, propertyId)"]
  RES --> SCOPE{"scope REFUSED?"}
  SCOPE -->|"yes"| DENY2["403"]
  SCOPE -->|"no"| OK2["allow / deny by grants"]
  PLAT -->|"no"| OPEN["allow any authenticated caller<br/>(FO-P1-001)"]

  classDef defect fill:#fee2e2,stroke:#dc2626,color:#991b1b
  classDef confirmed fill:#d1fae5,stroke:#059669,color:#065f46
  classDef unverified fill:#fef3c7,stroke:#d97706,color:#92400e
  class DENY1 defect
  class RES,SCOPE,OK2 confirmed
  class OPEN defect
```

**Confirmed:**
- Guard order/registration: `app.module.ts:107-109` (JWT, property scope, multi-tenant) + `common/authorization/authorization.module.ts:18-22` (roles, common permission, resource access).
- `request.user.permissions` is not hydrated (`auth.service.ts:129-140`), so the common `@Permissions` guard (§9.2) is effectively admin-only (`FO-P2-005`).
- The platform `@Permission` guard works and enforces scope (test-derived, `fo-route-permissions.spec.ts` PASS).
- Routes with neither decorator remain open to any authenticated caller (`FO-P1-001`).

---

## 6. No-Show Seam (defect)

```mermaid
flowchart LR
  A["ArrivalPanel 'No Show'"] --> B["frontOfficeApi.markNoShow(id)"]
  B --> C["PUT /reservations/:id<br/>{ reservation_status: 'NO-SHOW' }"]
  C --> D{"UpdateReservationDto<br/>declares reservation_status?"}
  D -->|"NO"| E["ValidationPipe forbidNonWhitelisted<br/>→ HTTP 400"]
  D -->|"would pass"| F["generic update (wrong owner)"]
  G["POST /reservations/:id/no-show"] --> H["ProcessNoShowCommand<br/>(never called)"]

  classDef defect fill:#fee2e2,stroke:#dc2626,color:#991b1b
  classDef unverified fill:#fef3c7,stroke:#d97706,color:#92400e
  class E,F,H defect
```

---

## 7. Status Vocabulary (shared, confirmed)

```mermaid
stateDiagram-v2
  [*] --> PROSPECT
  PROSPECT --> PENDING
  PROSPECT --> CONFIRMED
  PROSPECT --> WAITLIST
  PENDING --> CONFIRMED
  PENDING --> GUARANTEED
  PENDING --> CANCELLED
  PENDING --> WAITLIST
  WAITLIST --> CONFIRMED
  WAITLIST --> PENDING
  WAITLIST --> CANCELLED
  CONFIRMED --> GUARANTEED
  CONFIRMED --> CHECKED_IN
  CONFIRMED --> CANCELLED
  CONFIRMED --> NO_SHOW
  CONFIRMED --> CHECKED_OUT
  GUARANTEED --> CHECKED_IN
  GUARANTEED --> CANCELLED
  GUARANTEED --> NO_SHOW
  GUARANTEED --> CHECKED_OUT
  CHECKED_IN --> CHECKED_OUT
  CHECKED_IN --> CONFIRMED
  CHECKED_OUT --> CHECKED_IN
  CANCELLED --> CONFIRMED
  NO_SHOW --> CONFIRMED
```

Source: `packages/shared/src/reservation-state-machine.ts:32-42`; enforcement: `reservations/domain/services/reservation-status.service.ts:30-36`.

---

## 8. Coverage of the Map

| Seam | Status |
|---|---|
| Reservations → FO check-in deep link | Confirmed (source + E2E test exists, not executed) |
| Check-in staged flow identifier | Confirmed UUID |
| Check-in guest/shares identifier | Defect (`FO-P2-001`) |
| FO → Reservations extend / reinstate | Confirmed (test-derived) |
| FO → Reservations no-show | Defect (`FO-P2-002`) |
| Check-out status transition | Confirmed |
| Check-out financial ordering / idempotency | Defect (`FO-P2-003`) |
| Check-out settlement gate | Suspected (`FO-P2-008`) |
| Folio authority | Suspected / potential risk (`FO-P2-006`) |
| Room overlap safeguard | Unreachable (`FO-P2-007`) |
| Permission enforcement | Confirmed split + defect (`FO-P2-005`) |
| FE/BE permission vocabulary | Defect (`FO-P2-009`) |
| Realtime invalidation matrix | Partially verified (not exhaustive) |
| `apps/mobile` | Not scanned (carried from Phase 1) |

---

*End of Phase 2 integration map.*

**Document:** `12_API_CONTRACT.md`  
**System:** XYLO Enterprise PMS  
**Domain:** Reservations  
**Status:** API CONTRACT / IMPLEMENTATION STANDARD  
**Date:** 2026-08-26

---

# 1. Purpose

This document defines the API contract standards for the XYLO Reservations domain.

It establishes how the Reservation backend exposes:

- commands;
- queries;
- lifecycle operations;
- search;
- reservation details;
- availability and rate information;
- group-related operations;
- integrations;
- errors;
- pagination;
- authorization;
- concurrency;
- idempotency;
- auditability.

This document defines the **contract boundary** between consumers and the Reservations backend.

It does not make the HTTP API the domain model.

The domain remains authoritative.

---

# 2. Contract Authority

The API is an application boundary.

The dependency direction is:

```text
Frontend / External Consumer
          ↓
       API Contract
          ↓
    Application Layer
          ↓
      Domain Model
          ↓
      Persistence

The API must not expose database structure as its contract.

3. API Design Principle

The API must express business intent, especially for state-changing operations.

Prefer:

POST /api/v1/reservations/{reservationId}/confirm

over:

PATCH /api/v1/reservations/{reservationId}

{
  "status": "CONFIRMED"
}

when confirmation is a business operation with rules, authorization, audit and side effects.

4. Base Path

The canonical public API namespace is:

/api/v1

Reservations endpoints are rooted at:

/api/v1/reservations

Exact deployment prefixes may vary by gateway/environment, but the logical contract remains versioned.

5. Versioning

The API must be explicitly versioned.

Current target:

v1

Breaking contract changes require a controlled versioning/migration strategy.

Do not silently change the meaning of an existing field.

6. Resource Naming

Use plural resource names:

/reservations

not:

/reservation

Resource paths use stable identifiers.

Example:

/reservations/{reservationId}
7. Identifier Constraint

The API must expose the canonical XYLO Reservation identifier.

External provider identifiers must remain separate.

A reservation may contain references such as:

reservationId
confirmationNumber
externalReference
channelReference

These must not be conflated.

8. Property Context

Every property-scoped Reservation request must execute within the authenticated property/tenant context.

The client must not be trusted to select an arbitrary property merely by supplying:

hotelId
propertyId

in the request.

The backend must enforce property isolation.

9. Authentication

All protected Reservation endpoints require the platform authentication mechanism.

The API must reject unauthenticated access according to the platform security standard.

Authentication answers:

Who is making this request?

Authorization answers:

What is this caller allowed to do?

10. Authorization

Every Reservation mutation must pass the established permission framework.

Examples of permission concepts include:

reservation.read
reservation.create
reservation.update
reservation.confirm
reservation.guarantee
reservation.cancel
reservation.reinstate
reservation.no_show
reservation.extend
reservation.transfer
reservation.change_rate

The exact permission identifiers must follow the platform permission registry.

11. No Frontend-Only Authorization

Hiding a button is not authorization.

This is insufficient:

if (!canCancel) hideCancelButton()

The backend must independently enforce the permission.

12. Content Type

JSON is the default API representation:

Content-Type: application/json

File/document operations must use the approved upload mechanism rather than embedding large binary documents inside normal Reservation JSON payloads.

13. Request Correlation

Requests should support the platform correlation/request identifier.

Example:

X-Request-Id: <request-id>

The exact header name must follow the enterprise platform convention.

Correlation identifiers must propagate through relevant logs, events and downstream operations.

14. Idempotency

Retryable create/mutation operations that can result in duplicate business effects must support idempotency.

Preferred pattern:

Idempotency-Key: <unique-key>

The server must associate the key with the relevant operation scope.

15. Idempotency Semantics

For an idempotent request:

First request
    ↓
Business operation executes
    ↓
Result stored

A retry with the same valid idempotency key should return the same logical result rather than creating another reservation or repeating the business effect.

16. Idempotency Scope

An idempotency key must not be treated as globally unique without context.

Its effective scope should account for:

Property
Authenticated actor/client where required
Operation
Idempotency key

The exact persistence design is defined by the backend architecture.

17. Create Reservation

Canonical endpoint:

POST /api/v1/reservations

Purpose:

Create a new Reservation aggregate/business transaction.

The request must contain the minimum information required to create a valid reservation according to the domain model.

18. Create Reservation — Conceptual Request

Example:

{
  "guestId": "guest-id",
  "arrivalDate": "2026-09-10",
  "departureDate": "2026-09-15",
  "roomTypeId": "room-type-id",
  "ratePlanId": "rate-plan-id",
  "source": "DIRECT",
  "adults": 2,
  "children": 0
}

This is illustrative.

The final request DTO must be generated from the locked domain model and actual implementation requirements.

19. Create Reservation Rules

Creating a reservation must not be equivalent to blindly inserting a database row.

The operation must evaluate applicable:

guest/profile rules
property context
inventory/availability
rate rules
reservation rules
authorization
financial/commercial requirements

where applicable.

20. Create Reservation Response

Successful creation should return a stable Reservation representation.

Example:

{
  "data": {
    "id": "reservation-id",
    "confirmationNumber": "XYL123456",
    "status": "PENDING"
  }
}

The actual status depends on the business rules and creation channel.

21. Get Reservation

Canonical endpoint:

GET /api/v1/reservations/{reservationId}

Purpose:

Retrieve the authoritative Reservation representation accessible to the caller.

The response must not expose unrestricted database data.

22. Reservation Detail Representation

The detail response may contain:

Reservation identity
Confirmation number
Lifecycle status
Guest reference
Stay dates
Room/rate information
Source/channel
Group reference
Commercial references
Operational references
Relevant timestamps

Sensitive data must be returned only when required and authorized.

23. Search Reservations

Canonical endpoint:

GET /api/v1/reservations

Purpose:

Search/list Reservations using server-side filtering, sorting and pagination.

Example:

GET /api/v1/reservations?status=CONFIRMED&arrivalFrom=2026-09-01&arrivalTo=2026-09-30
24. Search Parameters

Supported filters should be explicit and typed.

Potential filters include:

status
confirmationNumber
guestId
guestName
arrivalFrom
arrivalTo
departureFrom
departureTo
roomTypeId
ratePlanId
groupId
source
channel
createdFrom
createdTo

Only approved filters should be exposed.

25. Search Pagination

Large result sets must be paginated.

Preferred conceptual response:

{
  "data": [],
  "meta": {
    "page": 1,
    "pageSize": 50,
    "total": 1250
  }
}

If cursor pagination is selected by the architecture, the contract must use cursor metadata instead.

Do not mix pagination models inconsistently across endpoints.

26. Search Sorting

Sorting must use an explicit allow-list.

Example:

sort=arrivalDate
sort=-arrivalDate

The backend must reject arbitrary database expressions.

27. Search Field Selection

If field selection is supported, it must use an explicit allow-list.

Do not expose arbitrary database column selection.

28. Confirm Reservation

Canonical endpoint:

POST /api/v1/reservations/{reservationId}/confirm

Purpose:

Execute the business operation that transitions the Reservation into the confirmed state when allowed.

The backend must enforce the state transition.

29. Guarantee Reservation

Canonical endpoint:

POST /api/v1/reservations/{reservationId}/guarantee

Purpose:

Apply the Reservation guarantee operation according to the commercial and lifecycle rules.

Guarantee must not be implemented as a direct status update.

30. Cancel Reservation

Canonical endpoint:

POST /api/v1/reservations/{reservationId}/cancel

Conceptual request:

{
  "reasonCode": "GUEST_REQUEST",
  "reasonText": "Guest requested cancellation"
}

Cancellation rules may include:

permission
cancellation policy
inventory release
financial consequences
audit
events

where applicable.

31. Reinstate Reservation

Canonical endpoint:

POST /api/v1/reservations/{reservationId}/reinstate

Purpose:

Restore a cancelled/no-show Reservation only when the domain state machine permits it.

Reinstatement must not directly overwrite the status field.

32. No-Show Reservation

Canonical endpoint:

POST /api/v1/reservations/{reservationId}/no-show

Purpose:

Apply the no-show business operation.

This operation may have consequences for:

inventory
charges
availability
audit
notifications
channel synchronization

according to the business rules.

33. Amend Reservation

Canonical endpoint:

PATCH /api/v1/reservations/{reservationId}

This endpoint is for permitted Reservation attribute amendments.

It must not become a generic unrestricted business-state mutation endpoint.

34. Amendable Fields

Examples may include:

Guest association
Stay dates
Room type
Rate reference
Occupancy
Source
Market/segment
Notes
Reservation metadata

The final list must be defined by the domain model.

35. State Changes Through Amend

The generic amendment endpoint must not bypass lifecycle commands.

Do not permit:

{
  "status": "CANCELLED"
}

as an unrestricted status mutation.

Use the appropriate business command endpoint.

36. Change Room / Room Type

Where supported, room/room-type changes should use an explicit business operation.

Conceptual endpoint:

POST /api/v1/reservations/{reservationId}/change-room-type

Request:

{
  "roomTypeId": "new-room-type-id"
}

The operation must evaluate availability and other applicable rules.

37. Change Rate

Authoritative endpoint (resolved T1, 2026-10-10 — the previous conceptual
POST /change-rate with a `ratePlanId` payload described no endpoint that exists
or can exist: reservations have no rate-plan identity, only `rate_code`):

PUT /api/v1/reservations/{reservationId}/rate

Authorization: `reservations:reservation:changeRate`, standard property scope.

Request:

{
  "rateCode": "BAR",
  "nightlyRate": 250,
  "totalRate": 750
}

Validation (ValidationPipe, whitelist + forbid-non-whitelisted):

- `rateCode` — required non-empty string. Unknown properties are rejected (400).
- `nightlyRate`, `totalRate` — optional numbers (the Reservations web client
  always sends both; they are advisory display quotes). They are validated but
  not persisted: the reservation stores only the selected `rate_code`, and
  monetary values are recomputed read-side from `rate_details` for the
  reservation's current rate code and stay dates. The Reservations API does not
  calculate rates (§44, §120: rate calculation belongs to Rates / Revenue).

Behavior:

- Reservation must exist (404 `Reservation {id} not found`) and be in
  `CONFIRMED` or `GUARANTEED` status; any other status returns 422 with code
  `BUSINESS_RULE_INVALID_STATUS`.
- Success response: `{ "success": true, "modified": true }`
  (or `"modified": false` when nothing applied).
- Persisted effect: `reservations.rate_code` updated; availability assertion is
  not re-keyed for rate-only changes; `room_rate` and `rate_details` are left to
  the Rates authority.

Rate-code selection for the client uses the companion read endpoint:

GET /api/v1/reservations/rate-codes

Authorization: `reservations:reservation:view`. Response: array of
`{ "code", "description", "rateType", "rateCategory" }` (falls back to active
rate-plan sources when the hotel has no `rate_codes` rows).

The Front Office in-house rate workflow is a separate operation and is NOT part
of this contract: `POST /api/v1/front-office/{reservationId}/change-rate`
with `{ "newRateCode", "reason?" }` reprices remaining nights, writes daily
elements, and posts billing adjustments.

38. Extend Reservation

Conceptual endpoint:

POST /api/v1/reservations/{reservationId}/extend

Request:

{
  "newDepartureDate": "2026-09-18"
}

The operation must evaluate:

availability
rate implications
stay rules
authorization
financial implications

where applicable.

39. Transfer Reservation (Cross-Property Transfer)

TransferReservation means Cross-Property Reservation Transfer: moving an existing reservation from one XYLO property to another within the same management group/tenant. This is defined by ADR-072.

Conceptual endpoint:

POST /api/v1/reservations/{reservationId}/transfer

Conceptual request:

{
  "targetPropertyId": "property-id",
  "reasonCode": "GUEST_REQUEST",
  "reasonText": "Guest relocating to另一个 property"
}

The operation must evaluate:
- cross-property authorization (enterprise/super-admin required per master spec §10.2)
- rate applicability at the target property
- availability at the target property
- inventory release at source property
- inventory allocation at target property
- guest continuity
- external reference continuity
- audit requirements

Conceptual response:

{
  "data": {
    "id": "reservation-id",
    "sourcePropertyId": "original-property",
    "targetPropertyId": "new-property",
    "status": "PENDING"
  }
}

The following are NOT Reservation Transfer (they are different operations):
- Room transfer (physical room move) → Front Office operation
- Room-type change → POST /reservations/{id}/change-room-type
- Rate change → PUT /reservations/{id}/rate
- Date change → PATCH /reservations/{id}
- Stay extension → POST /reservations/{id}/extend

Authorization: Cross-property authorization required (not standard property-scoped permission).

Idempotency: Required. Retry with same idempotency key must not create duplicate transfers.

Concurrency: Must detect concurrent modifications. If the reservation was modified since the client read it, reject with 409 Conflict.

Events: ReservationTransferred domain event emitted.

40. Walk-In

Walk-in is a Front Office-initiated Reservation creation entry point. This is defined by ADR-073.

Walk-in creates a Reservation through the authoritative Reservation domain. It is NOT a second reservation model.

The walk-in workflow spans two domains:

1. Front Office owns: operational initiation, guest processing, immediate check-in
2. Reservations owns: reservation aggregate, lifecycle, business rules

Front Office endpoint:

POST /api/v1/front-office/walk-in

This endpoint internally invokes the CreateReservation command and then proceeds with the Front Office check-in workflow.

The Reservation API does not need a separate walk-in endpoint. Walk-in uses the same CreateReservation command (POST /api/v1/reservations) as manual booking. The Front Office layer orchestrates the combined walk-in + check-in experience.

If the guest does not yet exist, the walk-in workflow invokes the Guest/Profile domain to create or resolve guest identity before reservation creation.

If no room is available, the walk-in may:
- fail entirely (no reservation created), or
- create a reservation in PENDING state pending room assignment

The implementation must define this behavior explicitly during Phase 1.

Authorization: Standard Front Office permissions plus reservation.create permission.

Idempotency: Required for reservation creation. The walk-in endpoint must support idempotency to prevent duplicate reservations from retries.

Events: ReservationCreated domain event emitted (same as any reservation creation).

40a. Activities & Events

Activities & Events is a separate bounded domain, NOT owned by Reservations. This is defined by ADR-074.

The Reservations workspace may include an Activities & Events tab for navigation convenience. This is a UI aggregation concern, not domain ownership.

Activity/Event API (separate domain):

GET    /api/v1/activities
POST   /api/v1/activities
GET    /api/v1/activities/{activityId}
PATCH  /api/v1/activities/{activityId}
POST   /api/v1/activities/{activityId}/confirm
POST   /api/v1/activities/{activityId}/cancel

The Reservation API may expose a reference to an activity booking in the reservation detail response:

{
  "data": {
    "id": "reservation-id",
    "linkedActivityBookings": [
      {
        "activityId": "activity-id",
        "activityType": "banquet",
        "status": "confirmed"
      }
    ]
  }
}

Reservations must NOT contain activity availability, activity pricing, or activity booking lifecycle logic. Activity inventory remains owned by the Activity/Event domain.

41. Check-In Boundary

Check-in belongs to Front Office.

Reservation API may provide the data/handoff required by Front Office.

It must not expose a second independent check-in implementation under Reservations.

42. Check-Out Boundary

Check-out belongs to Front Office.

Reservation API may provide reservation/stay context but must not duplicate the Front Office check-out workflow.

43. Availability

Availability is an Inventory/CRS concern.

If Reservations needs availability information, use the approved availability API/application interface.

Do not invent:

GET /reservations/availability

that independently calculates inventory unless explicitly defined as an approved Reservation-facing projection.

44. Rates

Reservation rate selection must use the authoritative Rates capability.

The Reservation API may return selected rate information, but it must not become the rate calculation engine.

45. Guest References

Reservation requests should reference canonical Guest/Profile identities.

Example:

{
  "guestId": "guest-id"
}

The Reservation API must not silently create duplicate guest master records.

46. Guest Creation Boundary

If the business workflow requires creating a guest during reservation creation, this should invoke the approved Guest/Profile capability or application boundary.

Do not duplicate Guest persistence logic in Reservation repositories.

47. Group Reservations

Group-related endpoints must preserve the distinction between:

Group / Block
Allocation
Member Reservation

A member reservation remains a Reservation.

Group membership must not create a second reservation lifecycle.

48. External Channel Booking

External channels should enter through an integration boundary.

Conceptually:

External Provider
      ↓
Integration Adapter
      ↓
Normalized Command
      ↓
Reservation Application
      ↓
Reservation Domain

The provider payload must not become the internal domain contract.

49. External Booking Endpoint

If an integration endpoint is exposed, it should be isolated from the ordinary user-facing Reservation API namespace where appropriate.

Example conceptual structure:

/api/v1/integrations/{provider}/reservations

The exact route depends on the integration architecture.

50. Webhooks

Webhook endpoints must support:

authentication/signature verification
validation
idempotency
property resolution
correlation
audit/logging
retry safety

A webhook must not directly mutate Reservation persistence outside the application/domain boundary.

51. API Response Envelope

The target API should use a consistent response structure.

Successful single-resource response:

{
  "data": {}
}

Collection:

{
  "data": [],
  "meta": {}
}

Error:

{
  "error": {
    "code": "RESERVATION_NOT_FOUND",
    "message": "Reservation was not found.",
    "details": {}
  }
}

The exact envelope should follow the enterprise API standard if one already exists.

52. Error Codes

Business errors must use stable machine-readable codes.

Examples:

RESERVATION_NOT_FOUND
RESERVATION_INVALID_STATE
RESERVATION_CANCEL_NOT_ALLOWED
RESERVATION_GUARANTEE_NOT_ALLOWED
RESERVATION_NO_SHOW_NOT_ALLOWED
RESERVATION_REINSTATE_NOT_ALLOWED
RESERVATION_CONFLICT
RESERVATION_DUPLICATE
RESERVATION_RATE_NOT_ALLOWED
RESERVATION_INVENTORY_UNAVAILABLE

These are conceptual identifiers and must be reconciled with the platform error registry.

53. HTTP Status Semantics

Use HTTP status codes consistently.

Typical mapping:

200 OK
201 Created
202 Accepted
204 No Content
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Entity
429 Too Many Requests
500 Internal Server Error

The exact status should reflect the actual failure semantics.

54. 400 vs 422

Use 400 Bad Request for malformed requests.

Use 422 Unprocessable Entity where the request is structurally valid but violates validation/business input rules, if this matches the enterprise API standard.

Do not use status codes randomly.

55. Conflict Handling

Use 409 Conflict for meaningful state/concurrency conflicts where appropriate.

Examples:

Reservation was modified by another user
Reservation cannot transition from current state
Idempotency key reused with different payload
Inventory conflict
56. Validation Errors

Validation responses should identify the affected field where practical.

Example:

{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed.",
    "details": {
      "departureDate": [
        "Departure date must be after arrival date."
      ]
    }
  }
}
57. Error Stability

Clients must depend on:

error.code

rather than parsing human-readable message text.

Messages may be localized or improved without changing machine behavior.

58. Sensitive Error Handling

Do not expose:

database SQL
stack traces
internal file paths
secrets
provider credentials
sensitive guest data

in production API responses.

59. Concurrency Contract

Mutation requests must account for concurrent modification.

Where the API uses optimistic concurrency, clients may send a version or equivalent precondition.

Conceptual example:

If-Match: "<reservation-version>"

The exact mechanism must follow the backend architecture.

60. Stale Update Behavior

If the client attempts to update stale reservation data, the backend must reject or safely reconcile it according to the concurrency strategy.

It must not silently overwrite newer data.

61. Transaction Boundary

Each synchronous business mutation must have a clearly defined application transaction boundary.

The API layer must not manually orchestrate partial database writes.

62. Domain Events

Successful business operations may emit domain events.

Examples:

ReservationCreated
ReservationConfirmed
ReservationGuaranteed
ReservationCancelled
ReservationReinstated
ReservationNoShow
ReservationAmended

Events must be generated from authoritative operations.

63. Event Publication

Reliable integration events must use the enterprise transactional outbox.

The API must not return success while hiding a failed required event publication if the business contract requires the event.

64. Asynchronous Operations

Operations that are intentionally asynchronous may return:

202 Accepted

with an operation/job reference.

Do not return synchronous success for work that has not actually completed unless the contract explicitly defines asynchronous semantics.

65. Query Consistency

Reservation detail immediately following a successful mutation should reflect the authoritative committed state.

If a read model is eventually consistent, that behavior must be explicit.

66. Query Authorization

Authorization applies to queries as well as mutations.

A user must not retrieve a Reservation merely because the identifier is known.

67. Data Minimization

List endpoints should return summary representations.

Do not return the full Reservation aggregate, guest profile and all related objects for every list row.

68. Detail Expansion

If related resources can be expanded, expansion must use an explicit allow-list.

Conceptual:

?include=guest,room,rate

Do not allow arbitrary database graph expansion.

69. N+1 Protection

Backend query handlers must avoid generating uncontrolled database queries per returned reservation.

70. Filtering Security

Filter parameters must not allow arbitrary SQL or database expressions.

All filters must be validated against known fields and operators.

71. Sorting Security

Sort fields must be allow-listed.

Do not concatenate user-provided sort expressions into SQL.

72. Pagination Security

Pagination parameters must have safe limits.

Example:

pageSize <= configured maximum

Clients must not be allowed to request unbounded Reservation datasets.

73. Date Query Semantics

Date filters must use hotel/property operational date semantics where the business concept is a hotel date.

Timestamp filters must explicitly represent timestamps.

74. Time Zone Contract

API timestamps should use a consistent representation, normally ISO 8601.

Example:

2026-09-10T14:30:00Z

Hotel-local business dates should remain date values when time-of-day is not part of the concept.

75. Monetary Representation

Money returned by the API must preserve authoritative precision.

Avoid binary floating-point values for authoritative monetary amounts.

A conceptual representation may be:

{
  "amount": "1250.00",
  "currency": "SAR"
}

The exact enterprise money type must be standardized across XYLO.

76. Null Semantics

The API must distinguish between:

field omitted
field explicitly null
field containing an empty value

especially for PATCH/amendment operations.

77. PATCH Semantics

If PATCH is used:

omitted field
    =
leave unchanged

An explicitly nullable field may mean:

set to null

This must be consistent across DTOs.

78. PUT Constraint

Do not use PUT merely because it is convenient.

If full replacement semantics are not supported, do not expose a misleading PUT endpoint.

79. Delete Constraint

Reservation deletion is not equivalent to cancellation.

If deletion exists for administrative/data-management purposes, it must have a separate permission and clearly defined semantics.

Normal operational cancellation should use:

POST /reservations/{id}/cancel
80. No Physical Delete by Default

Operational Reservations should not be physically deleted simply because they are cancelled.

Historical traceability must be preserved according to retention policy.

81. Audit Contract

State-changing endpoints must produce appropriate audit records.

Audit information should identify:

actor
property
reservation
operation
time
result

where required.

82. API Logging

API logs should include enough information to trace failures using:

request ID
correlation ID
reservation ID
operation

but must not expose sensitive information.

83. API Documentation

All public Reservation endpoints must have machine-readable API documentation.

The project should expose the approved OpenAPI documentation.

84. DTO Constraint

Request/response DTOs must not simply expose Prisma models.

Create explicit API contracts.

This protects the API from accidental database-schema changes.

85. Domain-to-DTO Mapping

Conceptually:

API Request DTO
      ↓
Command
      ↓
Domain/Application
      ↓
Result
      ↓
Response DTO

Mapping must be deliberate.

86. No Prisma Leakage

Do not return:

Prisma model instances

directly from controllers as the API contract.

87. Enum Contract

API enums must be stable.

Do not expose arbitrary database strings where the client requires a controlled enumeration.

Reservation status values must correspond to the canonical lifecycle model.

88. Reservation Status Contract

The API must use the canonical Reservation lifecycle statuses defined by the domain specification.

Current domain status vocabulary identified during the audit includes:

PROSPECT
PENDING
CONFIRMED
GUARANTEED
CHECKED_IN
CHECKED_OUT
CANCELLED
NO_SHOW
WAITLIST

The final authoritative status model is defined by the locked domain specification.

Do not add or remove statuses merely to simplify frontend code.

89. Status Transition Contract

Clients must not assume every status can transition to every other status.

The API is the authority for allowed transitions.

90. Command Naming

Business command endpoints should use clear verbs:

confirm
guarantee
cancel
reinstate
no-show
extend
transfer
change-room-type

Avoid ambiguous endpoints such as:

/process
/update-status
/action
/do
91. Command Response

A successful command should return the resulting authoritative representation where practical.

Example:

{
  "data": {
    "id": "reservation-id",
    "status": "CONFIRMED"
  }
}

This reduces stale frontend state.

92. Query/Command Separation

Reads should not mutate Reservation state.

Writes should not be hidden inside query endpoints.

Bad:

GET /reservations/{id}/refresh

if the operation changes business state.

93. Side Effects

Commands may cause side effects such as:

inventory changes
events
notifications
audit
channel synchronization

but these must follow the architecture.

The API contract should not hide important synchronous failure conditions.

94. External Side Effects

If a command triggers an external provider operation, the API must clearly define whether success means:

XYLO reservation committed

or:

XYLO reservation committed AND provider synchronization completed

These are different contracts.

95. Idempotent Cancellation

Cancellation should be designed with safe retry semantics.

A retry of the same cancellation command must not create duplicate cancellation records or contradictory inventory effects.

96. Idempotent No-Show

No-show operations must similarly protect against repeated execution.

97. Idempotent Confirmation

Confirmation requests must not create repeated side effects when retried.

98. Duplicate Create Protection

Reservation creation from an external source must have a defined duplicate-detection strategy.

Possible keys may include:

provider
externalBookingId
property
channel

depending on integration requirements.

99. API Compatibility

When replacing legacy Reservation endpoints:

Legacy endpoint
       ↓
Adapter / compatibility layer
       ↓
New application command

may be used temporarily.

Do not maintain two independent business implementations merely to preserve old routes.

100. Legacy Endpoint Removal

Every compatibility endpoint should have:

owner
migration target
consumer list
removal condition

It must not become permanent undocumented API surface.

101. Frontend Contract

The Reservations frontend must consume the API through the approved API client/data-access layer.

Components must not construct ad-hoc HTTP requests throughout the UI.

102. Frontend Error Mapping

The frontend should map stable error codes to user-facing messages.

Do not depend on parsing backend error strings.

103. Frontend Mutation Flow

Preferred:

User action
   ↓
UI mutation hook
   ↓
API client
   ↓
Command endpoint
   ↓
Backend validation
   ↓
Domain operation
   ↓
Response
   ↓
Cache update/invalidation
   ↓
UI
104. API and UX Constraint

The API must support the UX without forcing the frontend to reconstruct business logic.

If the UI requires:

canCancel
canConfirm
canReinstate

these may be exposed as authoritative capabilities when appropriate.

The frontend must not infer complex permission/state rules from raw fields alone.

105. Capability Representation

Where useful, reservation responses may expose allowed actions:

{
  "data": {
    "id": "reservation-id",
    "status": "CONFIRMED",
    "allowedActions": [
      "CANCEL",
      "AMEND"
    ]
  }
}

This is optional and must follow the final API architecture.

It must not replace backend authorization.

106. HATEOAS

Full HATEOAS is not required unless explicitly adopted by the platform.

Do not introduce it inconsistently.

107. Bulk APIs

Bulk Reservation operations require explicit contracts.

Example conceptual endpoint:

POST /api/v1/reservations/bulk/cancel

must define:

authorization
atomicity
partial failure
idempotency
maximum batch size
audit

before implementation.

108. Bulk Failure Representation

If partial success is allowed, the response must clearly distinguish:

successful items
failed items
reason

Do not return a generic success for a partially failed bulk operation.

109. Import APIs

Reservation imports must use a controlled import contract.

Imported records must still pass appropriate domain validation and duplicate detection.

110. Export APIs

Large Reservation exports should not synchronously dump unrestricted data through ordinary list endpoints.

Use an approved asynchronous export mechanism when necessary.

111. Security Headers

API deployment must follow the enterprise security/header standards.

Do not implement application-specific security headers inconsistently with the platform gateway.

112. Rate Limiting

Public/integration-facing endpoints should be subject to appropriate rate limiting.

Limits must not prevent normal hotel operations.

113. Observability

Important API operations must be observable through:

structured logs
metrics
traces where available
audit records

especially for:

create
confirm
guarantee
cancel
reinstate
no-show
amend
channel booking
114. Health vs Business APIs

Operational health endpoints must not be confused with Reservation APIs.

Do not expose internal health/debug information through Reservation endpoints.

115. API Testing Requirements

Each Reservation command requires tests for at least:

successful execution
invalid state
invalid input
unauthorized caller
wrong property
not found
concurrency conflict where applicable
idempotent retry
transaction failure where applicable
event behavior
116. Contract Testing

API contracts should be covered by automated tests so that frontend/integration consumers are not silently broken.

117. OpenAPI Requirements

OpenAPI definitions must accurately represent:

paths
parameters
request bodies
responses
errors
enums
authentication

Do not allow generated documentation to drift from the real API.

118. API Anti-Patterns

The following are prohibited:

PATCH status directly
Controller → Prisma business mutation
Frontend-only validation
Frontend-only authorization
Unbounded list endpoints
Unsafe dynamic SQL
Provider payload as domain model
Duplicate Reservation service
Duplicate Reservation database
Hidden state mutation in GET
Fake asynchronous success
Silent cross-property access
119. Canonical Reservation API Surface

The initial target surface is conceptually:

GET    /api/v1/reservations
POST   /api/v1/reservations
GET    /api/v1/reservations/{id}
GET    /api/v1/reservations/rate-codes
PATCH  /api/v1/reservations/{id}

POST   /api/v1/reservations/{id}/confirm
POST   /api/v1/reservations/{id}/guarantee
POST   /api/v1/reservations/{id}/cancel
POST   /api/v1/reservations/{id}/reinstate
POST   /api/v1/reservations/{id}/no-show
POST   /api/v1/reservations/{id}/extend
POST   /api/v1/reservations/{id}/transfer
PUT    /api/v1/reservations/{id}/rate
POST   /api/v1/reservations/{id}/change-room-type

This is the target contract shape, not a declaration that every endpoint currently exists.

Each endpoint must be verified against the implementation roadmap before coding.

120. API Ownership Matrix
API Capability	Authority
Reservation CRUD/read	Reservations
Reservation lifecycle	Reservations
Availability	Inventory / CRS
Rate calculation	Rates / Revenue
Guest master	Guests
Company master	Companies
Travel Agent master	Travel Agents
Check-in	Front Office
Check-out	Front Office
Room assignment	Front Office
Billing	Billing
Group/block	Group capability
External channel translation	Integration layer
UI state	Frontend
121. API Implementation Rule

Before adding an endpoint, answer:

1. Who owns this business capability?
2. Does an endpoint already exist?
3. Does another module already implement it?
4. Is this a command or a query?
5. What permission is required?
6. What transaction boundary applies?
7. What state transitions are possible?
8. Is idempotency required?
9. What events/audit are required?
10. What frontend/integration consumers exist?

If these answers are unknown, investigate before implementation.

122. API Change Procedure

For an API change:

Requirement
   ↓
Domain rule
   ↓
Application command/query
   ↓
API contract
   ↓
Implementation
   ↓
Tests
   ↓
OpenAPI
   ↓
Consumer migration
   ↓
Legacy removal

Do not begin with controller code and work backward.

123. API Contract and Database Independence

Database migrations must not automatically require API changes.

Likewise, API changes must not automatically expose new database fields.

The API is an intentional abstraction boundary.

124. API Contract and Domain Independence

Domain rules must not be weakened because a client cannot easily represent them.

The API should expose the correct business operation rather than forcing the domain into CRUD semantics.

125. API Contract and Legacy Code

Legacy APIs are evidence of the current system.

They are not automatically the target API.

Use 10_CODEBASE_MAP.md and 08_CURRENT_STATE.md to locate existing endpoints, then use the target architecture to determine which remain.

126. Final API Principle

The Reservation API must make it difficult to perform an invalid business operation accidentally.

The contract should guide consumers toward:

Explicit intent
   ↓
Validated command
   ↓
Authorized operation
   ↓
Atomic business transaction
   ↓
Auditable result
   ↓
Reliable events/integration
127. Final Rule

The API is not a thin CRUD wrapper around the Reservation tables.

It is the controlled application boundary of the XYLO Reservation domain.

Therefore:

The API must expose business capabilities, protect domain invariants, enforce property and permission boundaries, support reliable integrations, and prevent clients from bypassing the Reservation lifecycle.

128. Implementation Lock

Before implementing the final API surface, reconcile this document with:

01_DOMAIN_MODEL.md
03_PROCEDURES.md
04_BACKEND_ARCHITECTURE.md
05_FRONTEND_ARCHITECTURE.md
06_ROADMAP.md
07_DECISIONS.md
11_CONSTRAINTS.md

If a conflict is found, do not silently choose an implementation.

Record the conflict as an architectural decision and resolve it before locking the affected API contract

129. Sales Screen Coupon Validation - FND-12 Contract Decision

Status: DECIDED (B3.6, Phase 3; 2026-10-10). Decision: accept-and-ignore (document). Governing finding: FND-12 (phase-1/02_PHASE1_CLOSURE_AND_ROADMAP.md line 30; batch B3.6 line 85).

Request contract - POST /api/v1/reservations/validate-coupon

Required property: couponCode (string). The lookup is scoped to the caller's hotel and can succeed only for an existing coupon row (hotel_id + coupon_code) inside its validity window and usage limit. Rejections: 404 NOT_FOUND when no row matches (including an absent or unknown code), 422 BUSINESS_RULE_COUPON_EXPIRED when outside the validity window, 422 BUSINESS_RULE_COUPON_EXHAUSTED when the usage limit is reached. The sales-screen consumer additionally blocks empty input before sending (CouponValidator.tsx trim-guard line 26; disabled submit line 89).

The HTTP layer performs no field validation on this route: the controller binds an inline body type ({ couponCode: string }, reservations.controller.ts:530-535), and the global ValidationPipe (main.ts:28-41; whitelist + forbidNonWhitelisted) skips non-class metatypes, so no DTO, stripping, or 400 rejection applies here. Requiredness is enforced at the domain boundary, not the request boundary. An absent couponCode is serialized as a NULL query parameter and therefore matches no row (Prisma 5.22 raw-parameter serialization converts undefined to null and raises no client-side validation error on this path), producing the 404 above.

rateCode: accepted as an extra request property; NOT supported for rate-scoped coupon validation.

The frontend producer sends { couponCode, rateCode } (sales-screen.api.ts:26-33; use-coupon-validation.ts:8-16). The backend consumes couponCode only: the controller binds { couponCode } (reservations.controller.ts:533), the service forwards couponCode alone (reservations.service.ts:640-651), and the handler queries promotion_coupons by hotel_id + coupon_code with no rate predicate (validate-promotion-coupon.handler.ts:33-37). No rate-scoped coupon semantics exist in any authoritative source: no such entry in this contract or in 07_DECISIONS.md, and the promotion_coupons design carries no rate column (docs/Antigravity/v2/implementation_plan.md:1008-1022); the table is not represented in schema.prisma (raw-SQL access only). Extra request properties are tolerated on this route by current validation conventions (the inline body type bypasses the whitelist); that tolerance is compatibility, not product support.

Decision: accept-and-ignore (document). Rejected alternative: extend the contract to consume rateCode for rate-scoped validation - it would invent rate x coupon pricing semantics with no authoritative specification and no schema support. No backend, schema, or pipeline change is made by this decision.

Response contract - success (HTTP 201)

Canonical handler result, returned as the body with no envelope (the service unwraps via Result.getOrThrow, reservations.service.ts:650):

{ valid: true, couponCode, promotionCode, discountPercent?, discountAmount?, validFrom, validTo, remainingUses }

validFrom/validTo are Date values in the handler and serialize to ISO-8601 strings over HTTP.

Response contract - failure (AllExceptionsFilter envelope)

{ success: false, error: { code, message }, meta: { timestamp, requestId, ... } }

with codes 404 NOT_FOUND, 422 BUSINESS_RULE_COUPON_EXPIRED, and 422 BUSINESS_RULE_COUPON_EXHAUSTED as above.

Documented frontend divergences (latent; consumers unmounted as of B3.6)

- The FE type CouponValidation { valid, discount, expiryDate, usageCount } (sales-screen.api.ts:10-15) does not match the canonical response; its only consumer, CouponValidator.tsx, has no render sites in apps/web. If mounted, align the consumer to this contract first.
- The FE type DisplaySet.fields (sales-screen.api.ts:3-8) does not match the display-sets handler result (rateCodes, get-display-sets.handler.ts:12,82); its only consumer, DisplaySetsPanel.tsx, has no render sites.

display-sets confirmation (B3.6 acceptance evidence)

GET /reservations/display-sets is reachable and not shadowed by GET :id: apps/api/test/reservations-static-routes.e2e-spec.ts (the B2.4 supertest harness) passed 10/10 on 2026-10-10, including the display-sets reachability case and the @Permission / @PropertyScope locks.

Contract tests

apps/api/test/sales-payload-contract.e2e-spec.ts (HTTP boundary: FE payload accepted, rateCode not forwarded into business logic, canonical response passthrough; real-handler fixture for the canonical result and domain rejections) and apps/web/features/reservations/api/__tests__/b36-fnd12-sales-payload.test.ts (producer: exact URL and payload).

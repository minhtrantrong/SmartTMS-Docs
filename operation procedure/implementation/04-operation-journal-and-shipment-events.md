# Task 04 - Operation Journal and Shipment Events

## Objective

Implement the missing step `4` by adding a first-class operational event history that records business actions across intake, assignment, delivery, approvals, and accounting transitions.

## Dependencies

- Task 02

## Scope decision

Implement one generic `operation_journal` table and use filtered views or endpoints for shipment-specific timelines instead of creating multiple duplicate event tables.

## Deliverables

- Generic business event table.
- Automatic event emission for shipment and workflow actions.
- Query endpoints for global journal views and shipment timelines.
- FE timeline UI and read-only Mobile timeline support.

## Backend implementation

- Add `operation_journal` table with fields such as:
  - `id`, `entity_type`, `entity_id`, `event_code`
  - `actor_user_id`, `actor_role`
  - `occurred_at`, `correlation_id`
  - `summary`, `details_json`, `source`
- Create a shared event-writer service so handlers and domain services do not construct journal rows manually.
- Emit journal entries for at least:
  - order request submitted, reviewed, confirmed, rejected, converted
  - shipment created, assigned, status changed, bill uploaded
  - driver status reported
  - expense claim submitted and approved
  - statement sent and disputed
  - invoice issued and paid
- Add read endpoints:
  - global journal search and filtering
  - shipment timeline endpoint
  - entity-specific timeline endpoint if needed
- Ensure every event stores enough metadata to explain who did what and why.

## Suggested API surface

- `GET /api/operation-journal`
- `GET /api/operation-journal/:id`
- `GET /api/shipments/:id/events`

## Frontend implementation

- Add a reusable timeline component for shipment, order, statement, and invoice detail pages.
- Add a dispatcher or manager-facing operation journal view with filters by entity type, actor, date range, and event code.
- Surface important events inside existing shipment detail screens before adding a full journal workspace.

## Mobile implementation

- Add a read-only shipment timeline section to driver shipment detail if the screen already exists.
- Do not add a general audit console to mobile.

## Documentation updates

- Document standard event codes and minimum metadata fields.
- Document which business actions are required to emit journal events.

## Implemented event codes

### Order request

- `ORDER_REQUEST_SUBMITTED`
- `ORDER_REQUEST_REVIEWED`
- `ORDER_REQUEST_CONFIRMED`
- `ORDER_REQUEST_REJECTED`
- `ORDER_REQUEST_CONVERTED`

### Shipment

- `SHIPMENT_CREATED`
- `SHIPMENT_ASSIGNED`
- `SHIPMENT_STATUS_CHANGED`
- `SHIPMENT_BILL_UPLOADED`
- `SHIPMENT_CREATED_FROM_ORDER_REQUEST`

### Driver

- `DRIVER_STATUS_REPORTED`

## Minimum journal metadata

Every event must persist at least:

- `entity_type`
- `entity_id`
- `event_code`
- `occurred_at`
- `summary`
- `source`

When available, events should also populate:

- `actor_user_id`
- `actor_role`
- `correlation_id`
- `details_json`

## Required business-action emission map

- Order request submitted: emit `ORDER_REQUEST_SUBMITTED`.
- Order request review decision: emit one of `ORDER_REQUEST_REVIEWED`, `ORDER_REQUEST_CONFIRMED`, or `ORDER_REQUEST_REJECTED`.
- Order request conversion: emit `ORDER_REQUEST_CONVERTED` on the order request and `SHIPMENT_CREATED_FROM_ORDER_REQUEST` on the shipment.
- Shipment created from shipment CRUD: emit `SHIPMENT_CREATED`.
- Shipment assignment or reassignment: emit `SHIPMENT_ASSIGNED`.
- Shipment workflow transition: emit `SHIPMENT_STATUS_CHANGED`.
- Bill scan upload linked to a shipment: emit `SHIPMENT_BILL_UPLOADED`.
- Driver current-status update: emit `DRIVER_STATUS_REPORTED`.

## Current scope note

The current codebase does not yet contain expense-claim, statement, or invoice backend domains, so Task 04 now covers the operational journal for the workflows that already exist in code: order intake, shipment execution, driver status reporting, and shipment bill scanning. The accounting-oriented event families listed in the original task remain deferred until their source workflows are implemented.

## Implemented API surface

- `GET /api/operation-journal`
- `GET /api/operation-journal/:id`
- `GET /api/shipments/:id/events`
- `GET /api/order-requests/:id/events`

## Acceptance criteria

- Shipment and order lifecycles can be reconstructed from journal data.
- Journal entries record actor, timestamp, event type, and context payload.
- FE can display an entity timeline without deriving history from raw update timestamps.

## Implementation checklist

- [x] Create `operation_journal` migration.
- [x] Add event-writer service and repository.
- [x] Hook key business actions into the journal.
- [x] Add journal and shipment-events endpoints.
- [x] Add FE timeline components and a journal view.
- [x] Add automated tests for journal emission on core actions.
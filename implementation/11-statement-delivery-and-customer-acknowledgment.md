# Task 11 - Statement Delivery and Customer Acknowledgment

## Status

Implemented across backend and frontend.

## Objective

Implement step `8` by delivering statement packages to customers through a real customer lane and recording customer acknowledgment explicitly.

## Dependencies

- Task 01
- Task 03
- Task 10

## Deliverables

- Customer-accessible statement list and detail flow.
- Statement delivery tracking.
- Customer acknowledgment action.
- Customer-service assisted delivery flow for accounts not yet using self-service.

## Backend implementation

- Extend `shipment_statements` with delivery and acknowledgment fields or add a dedicated `statement_deliveries` table.
- Recommended delivery-tracking fields:
  - `statement_id`, `delivery_channel`, `sent_to_contact`, `sent_by_user_id`, `sent_at`
  - `opened_at`, `acknowledged_at`, `acknowledged_by_user_id`
- Add customer-scoped query endpoints that only expose the authenticated customer's own statements.
- Add service actions for:
  - mark statement as sent
  - customer acknowledge receipt
  - customer-service acknowledge on behalf of customer with audit note if required
- Emit journal events for send, open, acknowledge, and reopen actions.

## Suggested API surface

- `GET /api/customer-portal/statements`
- `GET /api/customer-portal/statements/:id`
- `POST /api/statements/:id/send`
- `POST /api/customer-portal/statements/:id/acknowledge`

## Frontend implementation

- Add customer portal statement list and statement detail pages.
- Add customer-service assisted send action in the internal workspace.
- Show delivery state, sent timestamp, and acknowledgment state on statement detail.
- Support a clear customer acknowledgment action with optional note.

## Mobile implementation

- No customer mobile delivery in this task.

## Documentation updates

- Document supported delivery channels and whether the customer portal is the primary channel or the canonical system of record.
- Document whether acknowledgment is mandatory before reconciliation opens.

## Implemented behavior

- Backend now persists statement delivery and acknowledgement metadata directly on `shipment_statements`, including `delivery_channel`, `sent_to_contact`, `sent_by_user_id`, `opened_at`, `acknowledged_at`, and `acknowledged_by_user_id`.
- Internal delivery actions are implemented with explicit routes:
  - `POST /api/statements/:id/send`
  - `POST /api/statements/:id/acknowledge`
- Customer self-service routes are now implemented under the customer portal:
  - `GET /api/customer-portal/statements`
  - `GET /api/customer-portal/statements/:id`
  - `POST /api/customer-portal/statements/:id/acknowledge`
- Customer portal detail access records the first customer open explicitly and emits a journal event.
- Statement send, open, acknowledge, dispute, reopen, and close actions remain auditable through the operation journal.

## Delivery channel policy

- Supported delivery channels in this task are:
  - `CUSTOMER_PORTAL`
  - `EMAIL`
  - `CUSTOMER_SERVICE_ASSISTED`
- The customer portal is the primary self-service delivery lane and the canonical system of record for customer-visible statement state in this task.
- Internal users can still record assisted delivery or email delivery for accounts that are not yet operating fully in self-service.

## Acknowledgement policy

- Customer acknowledgement is stored explicitly with actor and timestamp.
- Customer-service or other internal users can acknowledge on behalf of the customer only through the explicit assisted flow, and the backend requires an audit note for that action.
- Acknowledgement is not yet enforced as a prerequisite for reconciliation opening in the current implementation. Reconciliation gating remains a later workflow decision.

## Frontend surfaces

- Internal statement workspaces now show delivery state, send metadata, customer open state, acknowledgement state, and assisted acknowledgement controls.
- A dedicated customer-service statement-delivery workspace is available for assisted send and acknowledgement handling.
- Customer portal statement list and detail pages are now implemented with a self-service acknowledgement action.

## Acceptance criteria

- Customers can view only their own statements.
- Internal users can mark statements as sent and see delivery status.
- Customer acknowledgment is stored explicitly with actor and timestamp.
- Statement delivery and acknowledgment actions are auditable.

All acceptance criteria are implemented.

## Implementation checklist

- [x] Add statement delivery and acknowledgment persistence.
- [x] Add customer-scoped statement endpoints.
- [x] Add FE customer portal statement pages.
- [x] Add internal send action and state display.
- [x] Add access-control tests for customer statement visibility.
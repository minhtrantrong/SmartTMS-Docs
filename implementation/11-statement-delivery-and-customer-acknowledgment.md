# Task 11 - Statement Delivery and Customer Acknowledgment

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

## Acceptance criteria

- Customers can view only their own statements.
- Internal users can mark statements as sent and see delivery status.
- Customer acknowledgment is stored explicitly with actor and timestamp.
- Statement delivery and acknowledgment actions are auditable.

## Implementation checklist

- [ ] Add statement delivery and acknowledgment persistence.
- [ ] Add customer-scoped statement endpoints.
- [ ] Add FE customer portal statement pages.
- [ ] Add internal send action and state display.
- [ ] Add access-control tests for customer statement visibility.
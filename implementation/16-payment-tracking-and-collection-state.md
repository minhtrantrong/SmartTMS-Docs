# Task 16 - Payment Tracking and Collection State

## Objective

Complete the core accounting loop by tracking invoice payment state, collections, and outstanding balances after invoice issuance.

## Dependencies

- Task 13

## Deliverables

- Payment records linked to invoices.
- Invoice payment-state updates.
- Accountant collection tracking screen.
- Payment events for reporting and audit.

## Backend implementation

- Add tables:
  - `payments`
  - optional `payment_allocations` if one payment can cover multiple invoices
- Recommended payment fields:
  - `id`, `payment_number`, `invoice_id`
  - `payment_date`, `amount`, `currency`, `payment_method`
  - `reference_number`, `received_by_user_id`, `note`
  - `status`, `created_at`, `updated_at`
- Update invoice status logic so payments drive `PARTIALLY_PAID` and `PAID` transitions.
- If partial payments are allowed, recalculate outstanding balance server-side from recorded payments.
- Emit journal events when payments are recorded, reversed, or adjusted.

## Suggested API surface

- `POST /api/payments`
- `GET /api/payments`
- `GET /api/payments/:id`
- `POST /api/payments/:id/reverse`
- `GET /api/invoices/:id/payments`

## Frontend implementation

- Add accountant payment-entry and payment-history screens.
- Show invoice balance, last payment date, and collection status on invoice detail.
- Add filters for unpaid, partially paid, and paid invoices.

## Mobile implementation

- No payment administration on mobile.

## Documentation updates

- Document allowed payment methods and reversal policy.
- Document whether customer-visible invoice pages should show payment state.

## Acceptance criteria

- Users can record payments against invoices.
- Invoice balance and payment state update automatically.
- Payment history is preserved and auditable.
- Revenue reports can separate invoiced from collected amounts.

## Implementation checklist

- [ ] Create payment migration and model.
- [ ] Add invoice-balance recalculation logic.
- [ ] Add FE accountant payment screens.
- [ ] Add payment reversal handling.
- [ ] Add tests for partial and full payment scenarios.
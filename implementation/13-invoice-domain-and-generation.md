# Task 13 - Invoice Domain and Generation

## Objective

Implement step `10` by replacing shipment-total inference with a dedicated invoice domain generated from approved operational and reconciliation outputs.

## Dependencies

- Task 01
- Task 10
- Task 12

## Deliverables

- Invoice and invoice-line tables.
- Invoice numbering and issuance workflow.
- Invoice generation from reconciled statement data.
- Accountant workspace for invoice review and issue actions.

## Backend implementation

- Add tables:
  - `invoices`
  - `invoice_lines`
  - optional `invoice_documents` if generated PDF metadata must be stored
- Recommended invoice fields:
  - `id`, `invoice_number`, `customer_id`
  - `statement_id`, `status`, `issue_date`, `due_date`
  - `subtotal_amount`, `tax_amount`, `total_amount`, `currency`
  - `issued_by_user_id`, `void_reason`, `created_at`, `updated_at`
- Recommended invoice-line fields:
  - `invoice_id`, `shipment_id`, `statement_item_id`
  - `description`, `quantity`, `unit_price`, `line_amount`, `tax_amount`
- Add service rules:
  - invoice can only be generated from reconciled and approved statement outputs
  - invoice totals are recalculated server-side
  - voided invoices retain history and cannot be deleted silently
- Emit journal events for draft creation, issuance, void, and payment-state changes.

## Suggested API surface

- `POST /api/invoices/generate`
- `GET /api/invoices`
- `GET /api/invoices/:id`
- `POST /api/invoices/:id/issue`
- `POST /api/invoices/:id/void`

## Frontend implementation

- Add accountant invoice list, detail, and review pages.
- Show source statement references and line provenance so finance users can verify invoice lines.
- Add issue and void actions with confirmation and reason capture.

## Mobile implementation

- No invoice administration on mobile.

## Documentation updates

- Document invoice numbering scheme, due-date rules, and source-of-truth relationship with statements.
- Document whether tax is in scope now or should stay configurable for a later phase.

## Acceptance criteria

- Invoices are persisted in dedicated tables and not inferred from shipment totals.
- Invoice generation uses reconciled statement output as its source.
- Finance users can review, issue, and void invoices with full history.
- Invoice totals reconcile to their source statement lines.

## Implementation checklist

- [ ] Create invoice migrations and models.
- [ ] Add invoice-generation and issue services.
- [ ] Add FE accountant invoice workspace.
- [ ] Add validation to prevent invoicing unresolved reconciliations.
- [ ] Add tests for generation, issue, void, and amount recalculation.
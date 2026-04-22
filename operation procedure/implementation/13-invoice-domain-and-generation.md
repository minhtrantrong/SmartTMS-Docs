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

## Implementation notes

- Invoice numbers now use the runtime format `INV-YYYYMMDD-HHMMSS-######` so drafts and issued invoices share the same auditable sequence pattern.
- Invoice drafts are generated from persisted statement charge lines plus any statement-level adjustment line, and issue actions recalculate those totals server-side from the current statement output before the invoice moves to `ISSUED`.
- Due dates default from the customer payment terms at generation and issue time: `NET_30` falls back to 30 days, `NET_60` uses 60 days, and `COD` / `PREPAID` use the same day unless finance overrides the due date explicitly.
- Tax is stored on invoice headers and lines but remains `0` in this phase so tax policy can stay configurable in a later accounting phase without changing the invoice domain shape.

## Acceptance criteria

- Invoices are persisted in dedicated tables and not inferred from shipment totals.
- Invoice generation uses reconciled statement output as its source.
- Finance users can review, issue, and void invoices with full history.
- Invoice totals reconcile to their source statement lines.

## Implementation checklist

- [x] Create invoice migrations and models.
- [x] Add invoice-generation and issue services.
- [x] Add FE accountant invoice workspace.
- [x] Add validation to prevent invoicing unresolved reconciliations.
- [x] Add tests for generation, issue, void, and amount recalculation.
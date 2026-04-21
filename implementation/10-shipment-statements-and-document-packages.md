# Task 10 - Shipment Statements and Document Packages

## Objective

Implement step `7` by creating the statement package that groups shipment quantities, supporting documents, and bill evidence into an internal accounting-ready artifact.

## Dependencies

- Task 04
- Task 06

## Deliverables

- Statement domain for grouping completed shipment evidence.
- Statement items for quantity and charge lines.
- Document package support for bills and related attachments.
- FE statement generation and review flow.

## Backend implementation

- Add tables:
  - `shipment_statements`
  - `statement_items`
  - `statement_documents`
- Recommended statement fields:
  - `id`, `statement_number`, `customer_id`, `period_start`, `period_end`
  - `status`, `generated_by_user_id`, `generated_at`, `sent_at`, `closed_at`
  - `subtotal_amount`, `adjustment_amount`, `total_amount`, `notes`
- Recommended statement-item fields:
  - `shipment_id`, `description`, `quantity`, `unit_price`, `line_amount`
  - references to shipment, dispatch order, and later invoice line if invoiced
- Recommended document support:
  - link existing `shipment_bills` and uploaded evidence into a statement package
  - optionally store document type and display order in `statement_documents`
- Add generation logic that pulls only eligible shipments, for example delivered shipments with required evidence present.
- Emit journal events when a statement is generated, edited, sent, acknowledged, disputed, and closed.

## Suggested API surface

- `POST /api/statements/generate`
- `GET /api/statements`
- `GET /api/statements/:id`
- `PUT /api/statements/:id`
- `GET /api/statements/:id/documents`

## Frontend implementation

- Add internal statement-generation filters by customer and period.
- Show generated items, totals, linked shipments, and attached evidence in one review screen.
- Prevent statement finalization if required evidence is missing.

## Mobile implementation

- No dedicated statement-generation workflow on mobile.
- Driver mobile may later show whether required evidence for a shipment is complete, but that is not in scope for this task.

## AI impact

- If finance needs richer OCR fields than the current raw text payload, extend the AI response contract in a backward-compatible way. This is optional unless statement review cannot rely on existing OCR output.

## Documentation updates

- Document statement eligibility rules, required evidence, and statement numbering format.
- Document how statement totals relate to later invoice generation.

## Acceptance criteria

- Internal users can generate a statement package for eligible shipments.
- Each statement keeps itemized shipment references and attached supporting documents.
- Statement data is stored independently from invoices and reconciliation records.
- Missing document requirements block final statement completion.

## Implementation checklist

- [ ] Create statement migrations and models.
- [ ] Add statement-generation service.
- [ ] Link shipment evidence into statement documents.
- [ ] Add FE generation and review screens.
- [ ] Add eligibility and evidence validation.
- [ ] Add tests for statement totals and document completeness.
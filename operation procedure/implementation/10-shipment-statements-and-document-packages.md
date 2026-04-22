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

## Implemented rules

- Eligibility now uses customer-scoped shipments that are already `DELIVERED`, have an `actual_delivery_date` inside the selected reporting period, are not already referenced by another statement package, and have at least one linked `shipment_bills` record.
- Statement numbers follow the runtime format `STM-YYYYMMDD-HHMMSS-######` using the shared backend document-number pattern.
- Generated statement items include informational quantity rows plus accounting charge rows. The statement `subtotal_amount` is the sum of charge rows only, `adjustment_amount` is header-level, and `total_amount = subtotal_amount + adjustment_amount`.
- Statement documents are currently generated from existing shipment bill evidence. Each linked shipment bill becomes a `statement_documents` row with document type `SHIPMENT_BILL`, display ordering, and OCR payload available through the referenced bill record.
- Statement progression beyond `GENERATED` is blocked when any linked shipment lacks packaged bill evidence. The FE review workspace surfaces those gaps before users try to send or close the statement.
- Invoice generation is still out of scope. Statement items reserve `invoice_line_reference` for later invoice linkage without coupling the new statement domain to invoices today.

## Acceptance criteria

- Internal users can generate a statement package for eligible shipments.
- Each statement keeps itemized shipment references and attached supporting documents.
- Statement data is stored independently from invoices and reconciliation records.
- Missing document requirements block final statement completion.

## Implementation checklist

- [x] Create statement migrations and models.
- [x] Add statement-generation service.
- [x] Link shipment evidence into statement documents.
- [x] Add FE generation and review screens.
- [x] Add eligibility and evidence validation.
- [x] Add tests for statement totals and document completeness.
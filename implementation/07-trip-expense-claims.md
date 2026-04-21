# Task 07 - Trip Expense Claims

## Objective

Implement the first half of step `6.1` by giving drivers a formal way to submit trip expenses with receipts and reviewable line items.

## Dependencies

- Task 01
- Task 02
- Task 04

## Deliverables

- Trip expense claim domain.
- Trip expense item detail lines and attachments.
- Driver submission workflow.
- Manager and accountant visibility into submitted claims.

## Backend implementation

- Add tables:
  - `trip_expense_claims`
  - `trip_expense_items`
  - `trip_expense_attachments` or a reusable generic attachment table
- Recommended claim fields:
  - `id`, `claim_number`, `shipment_id`, `driver_id`
  - `status`, `currency`, `total_amount`, `submitted_at`
  - `submitted_by_user_id`, `review_note`, `created_at`, `updated_at`
- Recommended item fields:
  - `category`, `description`, `amount`, `expense_date`, `vendor_name`, `receipt_reference`
- Add categories at minimum for fuel, toll, parking, meal, repair, helper, and other.
- Add service logic that recalculates header totals from item rows instead of trusting client totals.
- Emit journal entries when claims are created, submitted, edited after rejection, and approved.

## Suggested API surface

- `POST /api/trip-expense-claims`
- `GET /api/trip-expense-claims`
- `GET /api/trip-expense-claims/:id`
- `PUT /api/trip-expense-claims/:id`
- `POST /api/trip-expense-claims/:id/submit`
- `POST /api/trip-expense-claims/:id/attachments`
- `GET /api/trip-expense-claims/my`

## Implemented workflow

- The backend now treats a trip expense claim as one claim per shipment and driver, enforced by a unique database constraint.
- Drivers can work in `DRAFT` and `REJECTED` states, upload receipts per line item, and submit only when every line has at least one attachment.
- Reviewer workflow supports `UNDER_REVIEW`, `APPROVED`, `REJECTED`, and `POSTED` through a dedicated review endpoint.
- Operation-journal events are emitted for create, update, resubmission, review transitions, and receipt uploads.

Additional implemented endpoints:

- `GET /api/trip-expense-claims/:id/events`
- `PATCH /api/trip-expense-claims/:id/review`
- `GET /api/trip-expense-claims/:id/attachments/:attachmentId/download`

## Frontend implementation

- Add review screens for dispatcher, manager, or accountant users depending on the approved ownership model.
- Show claim header, line items, receipts, and timeline in one place.
- Add filters for `DRAFT`, `SUBMITTED`, `UNDER_REVIEW`, `APPROVED`, `REJECTED`, and `POSTED`.

## Mobile implementation

- Add driver expense-claim creation flow linked to a completed or active trip.
- Support multiple line items and receipt uploads.
- Show current claim status and reviewer comments so drivers can correct rejected claims.

## Documentation updates

- Document valid expense categories and required evidence rules.
- Document whether a claim is per shipment, per dispatch order, or per trip segment. Recommended default: one claim per shipment or dispatch order.

## Documented decisions

- Valid expense categories: `FUEL`, `TOLL`, `PARKING`, `MEAL`, `REPAIR`, `HELPER`, `OTHER`.
- Evidence rule: every expense item must include at least one receipt attachment before the claim can be submitted.
- Claim granularity: the implemented default is one claim per shipment and driver.

## Acceptance criteria

- Drivers can create and submit expense claims with itemized amounts and receipt evidence.
- Backend recalculates totals and persists reviewer-visible detail lines.
- Reviewers can retrieve and filter claims without reading raw shipment cost fields.
- Claim events appear in the operation journal.

## Implementation checklist

- [x] Create expense-claim migrations and models.
- [x] Add item, attachment, and total-recalculation logic.
- [x] Add submission endpoints and validation.
- [x] Add FE reviewer screens.
- [x] Add Mobile driver submission flow.
- [x] Add tests for totals, validation, and journal emission.
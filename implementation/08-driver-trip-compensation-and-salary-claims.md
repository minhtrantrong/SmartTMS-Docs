# Task 08 - Driver Trip Compensation and Salary Claims

## Objective

Implement the second half of step `6.1` by creating a dedicated workflow for trip-based driver compensation and salary-related claims.

## Dependencies

- Task 01
- Task 02
- Task 04

## Deliverables

- Driver compensation claim domain separate from expense reimbursement.
- Support for trip pay, allowance, overtime, bonus, deduction, and notes.
- Reviewable compensation submissions that can later post into the accounting ledger.

## Backend implementation

- Add a `driver_trip_compensation` table with recommended fields:
  - `id`, `compensation_number`, `shipment_id`, `driver_id`
  - `status`, `base_trip_amount`, `allowance_amount`, `overtime_amount`
  - `bonus_amount`, `deduction_amount`, `claimed_total`
  - `compensation_note`, `submitted_at`, `submitted_by_user_id`
  - `approved_total`, `created_at`, `updated_at`
- If the business needs line-level detail, add `driver_trip_compensation_items` instead of keeping every amount on the header.
- Keep compensation records independent from payroll-close logic. This task is trip-claim capture only.
- Emit operation-journal events for draft, submit, approve, reject, and post transitions.

## Suggested API surface

- `POST /api/driver-trip-compensation`
- `GET /api/driver-trip-compensation`
- `GET /api/driver-trip-compensation/:id`
- `PUT /api/driver-trip-compensation/:id`
- `POST /api/driver-trip-compensation/:id/submit`
- `GET /api/driver-trip-compensation/my`

## Frontend implementation

- Add a review screen for managers or accountants to compare claimed amounts against trip facts such as route, duration, and shipment completion.
- Show compensation detail separate from expense reimbursement so reviewers do not mix operational spending with labor compensation.

## Mobile implementation

- Add driver-side compensation submission with structured fields for allowance, overtime, bonus requests, and explanation notes.
- Surface reviewer comments and approved total after review.

## Documentation updates

- Document which parts of compensation are driver-entered and which are policy-derived.
- Document whether compensation claims are mandatory before trip closure or optional for certain trip types.

## Implemented behavior

- Backend routes now live under `/api/driver-trip-compensation` with driver draft/create/update/submit actions, reviewer list/detail/review actions, and operation-journal timeline support.
- Driver compensation is stored separately from trip expense reimbursement and uses a header-level amount breakdown for base trip pay, allowance, overtime, bonus, deduction, claimed total, and approved total.
- Review screens are implemented separately from expense reimbursement so labor compensation review does not mix with operational spend review.
- Mobile now includes a dedicated driver compensation workspace with create, edit, submit, and review-result visibility.

## Compensation data ownership

- Driver-entered in this task: `base_trip_amount`, `allowance_amount`, `overtime_amount`, `bonus_amount`, `deduction_amount`, and `compensation_note`.
- Reviewer-entered in this task: `approved_total` and `review_note` during approval/rejection/posting.
- Policy-derived fields are not auto-calculated yet in Task 08. The claim stores the driver-submitted package first, and reviewers can adjust the approved total until later policy automation is introduced.

## Trip-closure policy

- Compensation claims are optional in the current implementation.
- Drivers can create compensation claims for their assigned non-cancelled trips; mobile surfaces the workflow for active or delivered trips without making it a prerequisite for trip closure.
- This keeps compensation capture independent from shipment master data and payroll-close logic, as required by the task.

## Acceptance criteria

- Drivers can submit compensation claims without overloading shipment master data.
- Reviewers can distinguish compensation from expense reimbursement.
- Approved totals can later feed accounting posting without re-keying.

## Implementation checklist

- [x] Create compensation migration and model.
- [x] Add claim submission and validation endpoints.
- [x] Add FE review screens.
- [x] Add Mobile submission screen.
- [x] Add journal emission and transition tests.
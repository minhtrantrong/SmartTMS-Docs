# Task 14 - Trip Cost Postings and Direct-Cost Ledger

## Objective

Implement the accounting side of step `12` by turning approved trip expense and compensation claims into authoritative trip cost postings.

## Dependencies

- Task 09

## Deliverables

- Direct-cost posting table.
- Posting flow from approved claims.
- Accountant visibility into posted and unposted claim records.
- Vehicle and shipment linkage for later reporting.

## Backend implementation

- Add `trip_cost_postings` table with recommended fields:
  - `id`, `posting_number`, `posting_date`, `status`
  - `source_type`, `source_id`
  - `shipment_id`, `dispatch_order_id`, `vehicle_id`, `driver_id`
  - `cost_category`, `amount`, `currency`, `memo`
  - `posted_by_user_id`, `posted_at`, `created_at`
- Support `source_type` values at minimum for:
  - `TRIP_EXPENSE_CLAIM`
  - `DRIVER_TRIP_COMPENSATION`
- Add service logic that creates postings only from approved source records and marks those records as `POSTED`.
- Prevent duplicate postings for the same approved source record.
- Emit journal events for post, reverse, and correction actions.

## Suggested API surface

- `POST /api/trip-cost-postings/post-approved`
- `GET /api/trip-cost-postings`
- `GET /api/trip-cost-postings/:id`
- `POST /api/trip-cost-postings/:id/reverse`

## Frontend implementation

- Add accountant screen to review which approved claims are still unposted.
- Show posting detail by source claim, shipment, vehicle, and cost category.
- Add reverse action with required reason capture.

## Mobile implementation

- No posting administration on mobile.

## Documentation updates

- Document which approved source types create direct-cost postings.
- Document reversal and correction policy.

## Implemented posting policy

- `TRIP_EXPENSE_CLAIM` creates one direct-cost posting per approved claim.
- Expense-claim postings use the approved claim total, keep shipment/driver linkage, and derive `cost_category` from the claim items.
- If all expense items share one category, that category is used; mixed-category claims are posted as `MIXED_TRIP_EXPENSE`.
- `DRIVER_TRIP_COMPENSATION` creates one direct-cost posting per approved compensation record.
- Compensation postings use `approved_total` when available, otherwise `claimed_total`, and use `DRIVER_TRIP_COMPENSATION` as the ledger `cost_category`.

## Reversal and correction policy

- Reversal requires a reason and changes the posting status to `REVERSED`.
- Reversal restores the originating source record from `POSTED` back to `APPROVED` so accounting can correct and repost it.
- Duplicate active postings are prevented per source record; only one `POSTED` trip-cost posting can exist for a given source at a time.
- Reposting a source after reversal is treated as a correction flow and is recorded with a correction journal event on the posting entity.

## Acceptance criteria

- Approved claims can be converted into dedicated trip cost postings.
- Each posting links back to the originating expense or compensation record.
- Duplicate posting is prevented.
- Posted records can feed vehicle financial reports later.

## Implementation checklist

- [ ] Create trip-cost-posting migration and model.
- [ ] Add posting and duplicate-prevention service logic.
- [ ] Add FE posting review screen.
- [ ] Add reverse or correction action.
- [ ] Add tests for posting idempotency and reversal.
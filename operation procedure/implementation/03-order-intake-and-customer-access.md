# Task 03 - Order Intake and Customer Access

## Objective

Implement the missing step `1` by introducing an explicit order-intake workflow that supports both internal customer-service intake and real external customer submission.

## Dependencies

- Task 01
- Task 02

## Scope decision

Use a dedicated `order_requests` domain instead of forcing pre-confirmation intake into `shipments`. A shipment should remain the execution object created after an order request is confirmed.

## Deliverables

- New `order_requests` table and backend CRUD flow.
- Customer-facing order submission and tracking lane.
- Internal customer-service review and conversion flow.
- Conversion from confirmed order request to shipment.

## Backend implementation

- Add new tables:
  - `order_requests`
  - optional `order_request_attachments` if intake documents are required
- Suggested `order_requests` fields:
  - `id`, `order_number`, `customer_id`, `requested_by_user_id`
  - contact and pickup/delivery fields
  - cargo summary fields
  - requested pickup and delivery dates
  - `status`, `review_notes`, `confirmed_at`, `converted_shipment_id`
  - `created_at`, `updated_at`
- Create repository, service, handler, and routes for:
  - customer create own order request
  - customer list own requests
  - customer-service review and confirm or reject
  - conversion to shipment
- Add service-layer conversion logic that creates a shipment from a confirmed order request and links the two records.
- Emit operation-journal events when an order is submitted, reviewed, confirmed, rejected, or converted.
- Restrict customers to their own records at query level, not only in UI.

## Suggested API surface

- `POST /api/order-requests`
- `GET /api/order-requests`
- `GET /api/order-requests/:id`
- `PATCH /api/order-requests/:id/review`
- `POST /api/order-requests/:id/convert-to-shipment`
- `GET /api/customer-portal/order-requests/me`

## Frontend implementation

- Add a customer-service workspace for order intake review and confirmation.
- Add a customer portal workspace for:
  - new order request form
  - order request detail
  - order request status list
- Extend the FE API client with order-request methods.
- Reuse shipment address and cargo form patterns where possible, but keep order-request DTOs separate from shipment DTOs.

## Mobile implementation

- No customer mobile delivery in this task.
- If driver mobile shows shipment origin context later, it should consume the converted shipment only, not raw order requests.

## Documentation updates

- Document the mapping from `order_requests` to `shipments`.
- Document who is allowed to submit, review, confirm, reject, and convert orders.

## Implemented access model

- `CUSTOMER` can submit new order requests and list or view only their own records.
- `CUSTOMER_SERVICE`, `MANAGER`, and `ADMIN` can create intake requests on behalf of a customer, review requests, confirm or reject them, and convert confirmed requests into shipments.
- Customer scoping is enforced in backend queries for list and detail access, not only in the frontend.

## Implemented mapping to shipments

- `order_requests.status` must be `CONFIRMED` before conversion is allowed.
- Converting an order request creates a new shipment with a generated `shipment_number`.
- The conversion copies these fields from `order_requests` into `shipments`:
  - `customer_id`
  - pickup and delivery address fields
  - `requested_pickup_date` -> `scheduled_pickup_date`
  - `requested_delivery_date` -> `scheduled_delivery_date`
  - cargo summary, package count, handling flags, priority, and special instructions
- The created shipment starts as an operational shipment with no route, driver, or vehicle assigned yet.
- After conversion, `order_requests.converted_shipment_id` is populated and the order request status becomes `CONVERTED_TO_SHIPMENT`.

## Implemented journal events

- `ORDER_REQUEST_SUBMITTED`
- `ORDER_REQUEST_REVIEWED`
- `ORDER_REQUEST_CONFIRMED`
- `ORDER_REQUEST_REJECTED`
- `ORDER_REQUEST_CONVERTED`
- `SHIPMENT_CREATED_FROM_ORDER_REQUEST`

## Acceptance criteria

- External customers can create and track their own order requests.
- Internal customer-service users can review, confirm, reject, and convert requests.
- Confirmed order requests can create shipments through a dedicated conversion action.
- Shipment creation for operational execution no longer depends only on internal generic shipment CRUD.

## Implementation checklist

- [x] Create `order_requests` migration and model.
- [x] Add backend CRUD and review endpoints.
- [x] Add conversion-to-shipment service logic.
- [x] Add FE customer-service review screens.
- [x] Add FE customer portal submission screens.
- [x] Add access-control tests for customer-scoped records.
# Task 06 - Dispatch Order Workflow

## Objective

Close step `5` by introducing a dedicated dispatch workflow instead of relying only on direct shipment edits for assignment and operational control.

## Dependencies

- Task 02
- Task 03
- Task 04

## Scope decision

Introduce a `dispatch_orders` domain linked to `shipments`. An order request becomes a shipment, and a shipment is then operationalized through a dispatch order.

## Deliverables

- New dispatch-order model and status lifecycle.
- Explicit assignment actions for driver and vehicle decisions.
- Dispatcher workspace for assignment, acknowledgment, and tracking.
- Driver acknowledgment of assigned work.

## Backend implementation

- Add `dispatch_orders` table with recommended fields:
  - `id`, `dispatch_number`, `shipment_id`
  - `assigned_driver_id`, `assigned_vehicle_id`
  - `status`, `dispatch_notes`, `assigned_at`, `acknowledged_at`
  - `assigned_by_user_id`, `cancel_reason`, `created_at`, `updated_at`
- Create service logic for:
  - create dispatch order for confirmed shipment
  - assign or reassign driver and vehicle
  - driver acknowledge dispatch
  - start execution and complete dispatch
  - cancel dispatch order with reason
- Keep shipment assignment fields synchronized from the dispatch-order workflow instead of letting multiple code paths update assignments independently.
- Emit operation-journal events for assignment, reassignment, acknowledgment, start, completion, and cancellation.

## Suggested API surface

- `POST /api/dispatch-orders`
- `GET /api/dispatch-orders`
- `GET /api/dispatch-orders/:id`
- `PATCH /api/dispatch-orders/:id/assign`
- `PATCH /api/dispatch-orders/:id/acknowledge`
- `PATCH /api/dispatch-orders/:id/start`
- `PATCH /api/dispatch-orders/:id/complete`
- `PATCH /api/dispatch-orders/:id/cancel`
- `GET /api/dispatch-orders/my`

## Frontend implementation

- Add a dispatcher workspace for dispatch-order queue management.
- Implement assignment forms that choose shipment, driver, and vehicle from valid availability lists.
- Show acknowledgment and execution state so dispatchers can see whether the driver has accepted the assignment.
- Link from shipment detail to dispatch-order detail and timeline.

## Mobile implementation

- Add driver-facing dispatch-order detail screen.
- Add explicit acknowledge action and show assignment notes.
- Surface current dispatch order separately from the generic shipment list if needed.

## Documentation updates

- Document the relationship among order request, shipment, and dispatch order.
- Document when dispatcher edits should happen on the dispatch order instead of directly on the shipment.

## Acceptance criteria

- Assignment actions happen through explicit dispatch-order endpoints.
- Driver and vehicle assignment history is visible and auditable.
- Drivers can acknowledge assigned work before trip execution starts.
- Dispatch state is visible in FE and Mobile.

## Implementation checklist

- [ ] Create `dispatch_orders` migration and model.
- [ ] Add assignment and acknowledgment service logic.
- [ ] Synchronize shipment assignment fields from dispatch actions.
- [ ] Add FE dispatch-order workspace.
- [ ] Add Mobile dispatch acknowledgment flow.
- [ ] Add journal and transition tests.
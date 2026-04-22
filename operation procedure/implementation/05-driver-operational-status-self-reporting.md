# Task 05 - Driver Operational Status Self-Reporting

## Objective

Close step `3` by making driver operational availability and trip-state reporting an explicit workflow rather than an incidental field update on the driver record.

## Dependencies

- Task 02
- Task 04

## Deliverables

- Driver self-report endpoint set.
- Status history table.
- FE and Mobile quick actions for status reporting.
- Journal events for every submitted driver status change.

## Backend implementation

- Keep `drivers.current_status` as the current snapshot, but add a `driver_status_updates` history table.
- Suggested `driver_status_updates` fields:
  - `id`, `driver_id`, `status`, `note`
  - `location_lat`, `location_lng`
  - `reported_at`, `reported_by_user_id`
- Add driver self-service endpoints using authenticated identity instead of free-form driver ID input.
- Validate status transitions according to Task 02. Example: prevent `ON_TRIP` if the driver has no active dispatch order.
- Update the current driver row and create a journal event in one service transaction.

## Suggested API surface

- `POST /api/drivers/me/status`
- `GET /api/drivers/me/status-history`
- `GET /api/drivers/:id/status-history`

## Frontend implementation

- Add driver dashboard quick actions for `AVAILABLE`, `ON_TRIP`, `OFF_DUTY`, and `ON_BREAK`.
- Show current status and recent history in the driver workspace.
- For dispatcher or manager views, surface the last reported timestamp and latest note.

## Mobile implementation

- Add a dedicated driver self-report card on the dashboard or shipment home screen.
- Allow optional note and location capture with every status change.
- Show server-side rejection reasons if a requested transition is not allowed.

## Documentation updates

- Define which statuses are self-reported versus system-derived.
- Define how dispatcher views should interpret driver states.

## Implementation notes

- Self-reported driver operational statuses are `AVAILABLE`, `ON_TRIP`, `OFF_DUTY`, and `ON_BREAK`.
- Shipment workflow statuses remain system-derived from the shipment lifecycle and are not replaced by driver self-reports.
- `drivers.current_status` remains the fast snapshot for reads, while `driver_status_updates` stores the accepted report history.
- Until a dispatch-order domain exists, the backend treats active shipment assignment as the proxy for an "active dispatch order".
- The current proxy uses shipment states `ASSIGNED`, `PICKED_UP`, and `IN_TRANSIT` to decide whether `ON_TRIP`, `AVAILABLE`, or `OFF_DUTY` are valid.
- Accepted status changes are written transactionally to the driver snapshot, the history table, and the operation journal.
- Web dispatcher/admin shipment views now surface the latest reported timestamp and note alongside the current driver status.
- Mobile self-reporting supports optional notes and optional manual latitude/longitude capture; backend validation errors are surfaced directly to the driver.

## Dispatcher interpretation

- Treat `drivers.current_status` as the latest self-reported operational state, not as a replacement for shipment progress.
- Use the latest note and timestamp to understand freshness and context before contacting a driver.
- If a driver has an active shipment assignment, `AVAILABLE` and `OFF_DUTY` reports are rejected by workflow validation.
- `ON_BREAK` is allowed during active trips and can be used to distinguish temporary pause from end-of-shift.

## Acceptance criteria

- Drivers can update their own operational status from FE and Mobile.
- Status history is persisted and queryable.
- Dispatcher views show the most recent reported driver state.
- Every status change appears in the operation journal.

## Implementation checklist

- [x] Create `driver_status_updates` migration.
- [x] Add driver self-service endpoints.
- [x] Add FE quick actions and status history.
- [x] Add Mobile quick actions and note capture.
- [x] Emit journal entries for every accepted status update.
- [x] Add transition-validation tests.
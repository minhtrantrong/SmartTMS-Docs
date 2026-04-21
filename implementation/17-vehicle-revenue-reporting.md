# Task 17 - Vehicle Revenue Reporting

## Objective

Implement step `11` by exposing per-vehicle revenue reporting based on issued invoices and reconciled operational data.

## Dependencies

- Task 12
- Task 13

## Deliverables

- Backend vehicle-revenue aggregation endpoint.
- Period-based filters and drill-down data.
- Frontend vehicle revenue report with export support.

## Backend implementation

- Add read-optimized query logic or a reporting view that aggregates revenue by vehicle and period.
- Revenue must trace back through invoice lines to statement items and then to shipments or dispatch orders tied to vehicles.
- Support filters at minimum for:
  - period start and end
  - customer
  - vehicle
  - route
- Recommended response fields:
  - `vehicle_id`, `vehicle_number`
  - `period_start`, `period_end`
  - `invoice_revenue`, `adjustment_amount`, `net_revenue`
  - shipment count, route count, and drill-down references
- Keep this endpoint read-only. Do not store report-only numbers in shipment rows.

## Suggested API surface

- `GET /api/reports/vehicle-revenue`
- `GET /api/reports/vehicle-revenue/:vehicleId`
- `GET /api/reports/vehicle-revenue/:vehicleId/drilldown`

## Frontend implementation

- Add a vehicle revenue report page for managers and accountants.
- Support period filters, export action, and drill-down into underlying invoices and shipments.
- Show clear handling for vehicles with no revenue in the selected period.

## Mobile implementation

- No major mobile work in this task.

## Documentation updates

- Document how revenue is attributed to a vehicle when a shipment changes vehicle mid-execution.
- Document whether adjustments reduce revenue in the same period or the next reporting period.

## Acceptance criteria

- Users can view vehicle revenue by period.
- Report totals reconcile to invoice lines and reconciled shipment outputs.
- Drill-down shows the source invoices and shipment references behind each vehicle total.

## Implementation checklist

- [ ] Implement vehicle-revenue aggregation query or view.
- [ ] Add report endpoints with filters.
- [ ] Add FE report page with drill-down and export.
- [ ] Add reconciliation tests between report totals and invoice data.
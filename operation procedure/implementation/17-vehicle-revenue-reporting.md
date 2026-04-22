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

## Implemented rules

- Backend now exposes read-only vehicle revenue reporting under `/api/reports/vehicle-revenue`, `/api/reports/vehicle-revenue/:vehicleId`, and `/api/reports/vehicle-revenue/:vehicleId/drilldown` for `ADMIN`, `MANAGER`, and `ACCOUNTANT` users.
- Report period filters apply to `invoices.issue_date`, so revenue is recognized in the period when the invoice is issued rather than when the shipment was delivered.
- Shipment-backed charge revenue follows the statement snapshot, not the current shipment row alone: the report uses the statement item's stored `dispatch_order_id` vehicle first and falls back to `shipments.vehicle_id` only when no dispatch-order vehicle was captured.
- When a shipment changes vehicles mid-execution, the report attributes revenue to the vehicle on the dispatch order referenced by the statement item that later produced the invoice line. This keeps the report aligned with the accounting snapshot that finance actually billed.
- Statement-level adjustment lines do not carry their own shipment or vehicle key, so the report allocates each adjustment proportionally across the invoice's shipment-backed charge lines by line amount. That preserves reconciliation to invoice totals while still allowing route and vehicle filters to work.
- Because adjustments are represented on the issued invoice itself, they reduce or increase revenue in the same reporting period as that invoice issue date, not in a later reporting period.
- Frontend report pages now live at `/accountant/pnl` and `/admin/pnl`, include customer/vehicle/route filters, export actions for summary and drilldown CSV, and a detail view with route, customer, invoice, and shipment-backed drilldown rows.

## Acceptance criteria

- Users can view vehicle revenue by period.
- Report totals reconcile to invoice lines and reconciled shipment outputs.
- Drill-down shows the source invoices and shipment references behind each vehicle total.

## Implementation checklist

- [x] Implement vehicle-revenue aggregation query or view.
- [x] Add report endpoints with filters.
- [x] Add FE report page with drill-down and export.
- [x] Add reconciliation tests between report totals and invoice data.
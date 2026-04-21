# Task 18 - Vehicle PnL Reporting

## Objective

Implement step `14` by combining vehicle revenue, direct trip costs, and fixed-cost allocations into a per-vehicle PnL report.

## Dependencies

- Task 14
- Task 15
- Task 17

## Deliverables

- Backend PnL aggregation endpoint.
- Period-based vehicle PnL report with drill-down.
- Clear separation of revenue, direct cost, fixed cost, gross margin, and net contribution.

## Backend implementation

- Add reporting query logic or reporting views that combine:
  - vehicle revenue from Task 17
  - trip cost postings from Task 14
  - fixed-cost allocations from Task 15
- Recommended response fields:
  - `vehicle_id`, `vehicle_number`
  - `period_start`, `period_end`
  - `revenue_total`
  - `direct_cost_total`
  - `gross_margin`
  - `fixed_cost_allocated`
  - `net_contribution`
  - drill-down references for invoice lines, trip-cost postings, and allocation records
- If reporting periods can be locked, add optional persisted snapshots at period close. If period close is not yet implemented, keep the first release query-based.
- Keep the PnL endpoint read-only and sourced from authoritative ledgers.

## Suggested API surface

- `GET /api/reports/vehicle-pnl`
- `GET /api/reports/vehicle-pnl/:vehicleId`
- `GET /api/reports/vehicle-pnl/:vehicleId/drilldown`

## Frontend implementation

- Add a vehicle PnL report page for managers and accountants.
- Show summary tiles and tabular results by period.
- Add drill-down by vehicle to inspect revenue lines, direct cost postings, and fixed-cost allocation sources.
- Add export support aligned with the revenue report.

## Mobile implementation

- No new mobile work in this task.

## Documentation updates

- Document the PnL formula and period treatment rules.
- Document how negative revenue or reversal postings affect the report.

## Acceptance criteria

- Users can view vehicle PnL by period.
- Report values reconcile to invoice revenue, trip cost postings, and fixed-cost allocation records.
- Users can drill down from the summary to the contributing records.

## Implementation checklist

- [ ] Implement vehicle PnL aggregation query or view.
- [ ] Add filtered PnL endpoints.
- [ ] Add FE PnL page with drill-down and export.
- [ ] Add reconciliation tests for revenue, direct cost, and fixed-cost totals.
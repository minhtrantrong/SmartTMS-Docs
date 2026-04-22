# Task 15 - Fixed-Cost Ledger and Allocation Rules

## Objective

Implement task `15` by capturing fixed costs separately from trip costs and defining the allocation rules needed for vehicle-level profitability.

## Dependencies

- Task 01
- Task 02

## Deliverables

- Fixed-cost ledger.
- Allocation-rule configuration.
- Allocation run results by period and vehicle.
- Accountant UI for fixed-cost entry and allocation management.

## Backend implementation

- Add tables:
  - `fixed_costs`
  - `allocation_rules`
  - `allocation_runs`
  - `cost_allocations`
- Recommended fixed-cost fields:
  - `id`, `cost_number`, `cost_period`, `category`, `description`
  - `amount`, `currency`, `vendor_name`, `status`
  - `entered_by_user_id`, `created_at`, `updated_at`
- Recommended allocation-rule fields:
  - `id`, `rule_name`, `rule_basis`, `target_scope`, `is_active`
  - `effective_from`, `effective_to`, `configuration_json`
- Recommended allocation bases:
  - active vehicle count
  - mileage
  - completed trips
  - active days in period
- Add services to run allocation for a period and persist results per vehicle.
- Emit journal events for fixed-cost entry, allocation run, and allocation reversal.
- Persist allocation explanations per row in JSON so downstream profitability screens can explain every amount.

## Implemented policy

- Approved allocation methods:
  - active vehicle count
  - mileage
  - completed trips
  - active days in period
- Rule-selection policy:
  - accountants create named rules with one approved basis and the `VEHICLE` target scope
  - only active rules can be executed
  - optional effective-from and effective-to windows prevent out-of-policy runs for earlier or later periods
- Recalculation policy:
  - rerun is allowed for the same rule and period
  - the new run is stored as `COMPLETED`
  - the prior completed run is marked `SUPERSEDED`
  - prior allocation rows are marked `REVERSED`
  - both the supersede event and the replacement run are written to the operation journal

## Suggested API surface

- `POST /api/fixed-costs`
- `GET /api/fixed-costs`
- `PUT /api/fixed-costs/:id`
- `POST /api/allocation-rules`
- `GET /api/allocation-rules`
- `POST /api/cost-allocations/run`
- `GET /api/cost-allocations`

## Frontend implementation

- Add accountant pages for fixed-cost entry, rule configuration, and allocation-run review.
- Show how each allocation result was calculated so the later PnL screen can explain fixed-cost load by vehicle.

## Mobile implementation

- No fixed-cost administration on mobile.

## Documentation updates

- Document approved allocation methods and the rule-selection policy.
- Document whether allocation runs are recalculable after period close.

## Acceptance criteria

- Fixed costs are stored separately from direct trip costs.
- Users can define and run allocation rules by period.
- Allocation results are persisted by vehicle and period.
- The system can explain which rule produced each allocated amount.
- A rerun of the same rule and period supersedes the prior completed run and reverses its allocation rows.

## Implementation checklist

- [x] Create fixed-cost, rule, allocation-run, and allocation migrations.
- [x] Add allocation-run service logic.
- [x] Add FE accountant screens for fixed costs, rule management, and allocation review.
- [x] Persist allocation results by vehicle and period.
- [x] Add tests for rule execution and repeatability.
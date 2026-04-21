# Task 01 - Role Model and Permission Matrix

## Objective

Add the missing business personas required by the operation procedure and expand the permission model so later workflow tasks can be implemented without reworking RBAC again.

## Dependencies

- None

## Deliverables

- New `CUSTOMER` role for external customer access.
- New `ACCOUNTANT` role for finance and reporting workflows.
- Expanded permission catalog for approvals, statements, reconciliation, invoices, costs, payments, and PnL reporting.
- Updated seed data so local development can exercise the new roles immediately.

## Backend implementation

- Add a new migration after `000002_seed_data.up.sql` to alter the `user_role` enum and include `CUSTOMER` and `ACCOUNTANT`.
- Update `SmartTMS-BE/internal/models/models.go` so `UserRole` includes the new constants.
- Add new permissions to the seed data or a dedicated seed migration. Recommended permission groups:
  - `APPROVAL_READ`, `APPROVAL_WRITE`
  - `STATEMENT_READ`, `STATEMENT_WRITE`
  - `RECONCILIATION_READ`, `RECONCILIATION_WRITE`
  - `INVOICE_READ`, `INVOICE_WRITE`
  - `PAYMENT_READ`, `PAYMENT_WRITE`
  - `TRIP_COST_READ`, `TRIP_COST_WRITE`
  - `FIXED_COST_READ`, `FIXED_COST_WRITE`
  - `PNL_VIEW`, `PNL_EXPORT`
- Update role-permission assignments in the seed migration:
  - `CUSTOMER` should only see its own order, statement, reconciliation, and invoice records.
  - `ACCOUNTANT` should own finance, payment, statement finalization, and reporting permissions.
  - `CUSTOMER_SERVICE` remains an internal role and should not inherit finance write permissions by default.
- Review `SmartTMS-BE/internal/routes/routes.go` and replace hard-coded role assumptions where the new roles must be admitted later.
- Ensure login and current-user responses continue returning role and permission data without breaking existing clients.

## Frontend implementation

- Extend `SmartTMS-FE/src/types/index.ts` role union to include `CUSTOMER` and `ACCOUNTANT`.
- Update login redirect logic and route guards so unsupported roles do not fall into the wrong workspace.
- Add role-aware navigation entries for future customer and accountant workspaces. Placeholder routes are acceptable in this task.
- Verify that existing admin, dispatcher, driver, and customer-service navigation still renders correctly after the role expansion.

## Mobile implementation

- Update role parsing and auth-session handling so login does not fail when the backend returns `CUSTOMER` or `ACCOUNTANT`.
- Keep unsupported roles blocked from mobile-only operational screens until dedicated mobile scope is defined.
- Do not build customer or accountant mobile UI in this task.

## Documentation updates

- Update the role matrix in `SmartTMS-Docs/operation-procedure-implement-plan.md` only if the approved role names differ from the plan.
- Add a short role-permission matrix document or appendix under `SmartTMS-Docs` so implementation teams can reference the canonical mapping.

## Acceptance criteria

- Backend accepts and persists users with roles `CUSTOMER` and `ACCOUNTANT`.
- Seed data creates at least one development user for each new role.
- Login responses and route guards work for the expanded role set.
- FE and Mobile can deserialize and store sessions for the new roles without runtime errors.

## Implementation checklist

- [ ] Create the enum migration for `user_role`.
- [ ] Add new role constants in backend models.
- [ ] Add finance and approval permissions.
- [ ] Seed role-permission mappings for `CUSTOMER` and `ACCOUNTANT`.
- [ ] Update FE role union and route guard logic.
- [ ] Update Mobile role parsing.
- [ ] Document the approved role-permission matrix.
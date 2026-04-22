# Task 12 - Reconciliation Disputes and Adjustments

## Status

Implemented across backend and frontend. Mobile remains intentionally out of scope.

## Objective

Implement step `9` by adding a formal reconciliation workflow for quantity mismatches, missing documents, disputed charges, and approved adjustments.

## Dependencies

- Task 09
- Task 10
- Task 11

## Deliverables

- Reconciliation case domain.
- Adjustment records linked to statements and statement items.
- Customer-service and accountant work queue for dispute resolution.
- Explicit resolution states and audit trail.

## Backend implementation

- Add tables:
  - `customer_reconciliations`
  - `reconciliation_adjustments`
  - optional `reconciliation_comments` if comment history should not rely only on the journal
- Recommended reconciliation fields:
  - `id`, `statement_id`, `customer_id`, `status`
  - `opened_by_user_id`, `opened_at`, `resolved_at`, `resolution_summary`
- Recommended adjustment fields:
  - `reconciliation_id`, `statement_item_id`, `adjustment_type`
  - `original_value`, `proposed_value`, `approved_value`, `reason`, `status`
- Add service actions for:
  - open dispute
  - add adjustment proposal
  - approve or reject adjustment
  - resolve or escalate reconciliation
- Ensure invoice generation later only reads reconciled outputs, not unreconciled draft statement values.
- Emit journal events for dispute creation, adjustment proposal, approval, rejection, resolution, and escalation.

## Suggested API surface

- `POST /api/reconciliations`
- `GET /api/reconciliations`
- `GET /api/reconciliations/:id`
- `POST /api/reconciliations/:id/adjustments`
- `POST /api/reconciliations/:id/resolve`
- `POST /api/reconciliations/:id/escalate`

## Frontend implementation

- Add reconciliation queue screens for customer service and accountants.
- Show disputed statement items, supporting documents, proposed adjustments, and current status in one detail view.
- Allow users to accept, reject, or edit adjustment proposals based on permissions.

## Mobile implementation

- No reconciliation administration on mobile.

## Documentation updates

- Document allowed dispute types and who can approve an adjustment.
- Document whether reconciliation is required for all statements or only for disputed accounts.

## Implemented workflow

- Backend now models formal statement disputes through `customer_reconciliations` and `reconciliation_adjustments`.
- Reconciliation cases are opened against a delivered statement and automatically move the linked statement into `DISPUTED` status.
- Adjustment proposals are stored separately from statement items until an accountant-approved or manager-approved decision is recorded.
- Resolving a reconciliation applies only approved adjustments back to the statement and returns the statement to `SENT` or `ACKNOWLEDGED` state depending on whether customer acknowledgement had already been recorded.
- Unresolved reconciliation cases now block statement closure, which is the current invoice-ready proxy until a dedicated invoice domain exists.

## Implemented API surface

- `POST /api/reconciliations`
- `GET /api/reconciliations`
- `GET /api/reconciliations/:id`
- `GET /api/reconciliations/:id/events`
- `POST /api/reconciliations/:id/adjustments`
- `POST /api/reconciliations/:id/adjustments/:adjustmentId/approve`
- `POST /api/reconciliations/:id/adjustments/:adjustmentId/reject`
- `POST /api/reconciliations/:id/resolve`
- `POST /api/reconciliations/:id/escalate`

Customers, customer service, accountants, managers, and admins use the same reconciliation API. Customer access is automatically scoped to reconciliations belonging to the authenticated customer account.

## Frontend surfaces

- Customer portal now has a dedicated reconciliation workspace at `/customer/reconciliation`.
- Customer statement detail now includes a direct action into the reconciliation workspace for dispute opening.
- Customer service has an internal reconciliation queue at `/customer-service/reconciliation` for intake, proposal capture, escalation, and assisted resolution work.
- Accountants have an internal reconciliation queue at `/accountant/reconciliation` for proposal review, approval/rejection, and final financial resolution.
- Admins and managers have an oversight queue at `/admin/reconciliation` for escalations and exception handling.
- Reconciliation detail views show the disputed statement, statement items, packaged supporting documents, explicit adjustment proposals, and the operation-journal timeline in one place.

## Allowed dispute types

- `DISPUTED_CHARGE`
- `QUANTITY_MISMATCH`
- `MISSING_DOCUMENT`
- `OTHER`

## Adjustment approval ownership

- `ACCOUNTANT` can approve or reject adjustment proposals.
- `MANAGER` and `ADMIN` can also approve or reject adjustment proposals for oversight and exception handling.
- `CUSTOMER_SERVICE` can open cases, propose adjustments, escalate cases, and resolve non-financial coordination work, but cannot approve or reject adjustment values.
- `CUSTOMER` can open cases and add proposal context only for their own statements.

## Reconciliation requirement policy

- Reconciliation is not required for every statement.
- Reconciliation is only required for statements that have an active dispute case.
- Because invoice generation is still not implemented, the current downstream guard is statement closure: a statement cannot be closed while any linked reconciliation case remains unresolved.

## Acceptance criteria

- Customers or internal teams can open a reconciliation case against a statement.
- Adjustment proposals are tracked explicitly and tied to the affected statement items.
- Reconciliation cases have auditable lifecycle states until closure.
- Invoice generation can exclude unresolved reconciliation cases.

All acceptance criteria are implemented.

## Validation

- Backend focused validation passed with `go test ./internal/reconciliationtests`.
- Frontend reconciliation files are clean in VS Code diagnostics.
- Full frontend `npx tsc --noEmit` still reports the pre-existing unrelated error in `src/app/api/proxy/route.ts` due duplicate `headers` declarations; the new reconciliation files did not introduce additional TypeScript errors.

## Implementation checklist

- [x] Create reconciliation and adjustment migrations.
- [x] Add dispute and resolution services.
- [x] Add FE reconciliation queue and detail screens.
- [x] Add rules to block downstream invoicing for unresolved cases.
- [x] Add tests for dispute, adjustment, resolve, and escalate flows.
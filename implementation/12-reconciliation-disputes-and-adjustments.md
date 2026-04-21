# Task 12 - Reconciliation Disputes and Adjustments

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

## Acceptance criteria

- Customers or internal teams can open a reconciliation case against a statement.
- Adjustment proposals are tracked explicitly and tied to the affected statement items.
- Reconciliation cases have auditable lifecycle states until closure.
- Invoice generation can exclude unresolved reconciliation cases.

## Implementation checklist

- [ ] Create reconciliation and adjustment migrations.
- [ ] Add dispute and resolution services.
- [ ] Add FE reconciliation queue and detail screens.
- [ ] Add rules to block downstream invoicing for unresolved cases.
- [ ] Add tests for dispute, adjustment, resolve, and escalate flows.
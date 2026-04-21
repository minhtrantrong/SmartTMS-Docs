# Task 09 - Approval Workflow and Audit Actions

## Objective

Implement step `6.2` by creating a reusable approval queue and action history for trip expense and compensation workflows.

## Dependencies

- Task 07
- Task 08

## Scope decision

Implement a generic approval domain that can support multiple entity types instead of hard-coding approval logic inside each claim table.

## Deliverables

- Generic approval-request and approval-action tables.
- Manager and accountant approval queue.
- Approval, rejection, and request-changes actions with comments.
- Audit trail tied into the operation journal.

## Backend implementation

- Add tables:
  - `approval_requests`
  - `approval_actions`
- Recommended approval-request fields:
  - `id`, `entity_type`, `entity_id`, `status`
  - `current_assignee_role`, `current_assignee_user_id`
  - `submitted_by_user_id`, `submitted_at`, `resolved_at`
- Recommended approval-action fields:
  - `id`, `approval_request_id`, `action`, `actor_user_id`, `comment`, `created_at`
- Support at least these entity types:
  - `TRIP_EXPENSE_CLAIM`
  - `DRIVER_TRIP_COMPENSATION`
- Add service methods for:
  - submit for approval
  - approve
  - reject
  - request changes
  - resubmit after correction
- Keep the source domain status and the approval-request status synchronized through one service layer.
- Emit both approval-action history and operation-journal entries.

## Suggested API surface

- `GET /api/approvals/queue`
- `GET /api/approvals/:id`
- `POST /api/approvals/:id/approve`
- `POST /api/approvals/:id/reject`
- `POST /api/approvals/:id/request-changes`
- `POST /api/approvals/:id/resubmit`

## Frontend implementation

- Add an approval queue workspace for managers and accountants.
- Show source entity details inline so reviewers do not need to navigate away for every decision.
- Show full action history including actor, timestamp, and comment.

## Mobile implementation

- No approval administration on mobile.
- Driver mobile must show current approval status, reviewer comments, and whether resubmission is required.

## Documentation updates

- Document approval ownership and escalation path.
- Document which roles can approve which entity types.

## Acceptance criteria

- Expense and compensation claims can be submitted into a centralized approval queue.
- Every approval decision records actor, timestamp, and comment.
- Rejected or change-requested claims can be corrected and resubmitted.
- Approval status is visible from both source records and the queue.

## Implementation checklist

- [ ] Create approval migrations and models.
- [ ] Add generic approval services and handlers.
- [ ] Link expense and compensation domains to approval requests.
- [ ] Add FE approval queue.
- [ ] Add Mobile status visibility for submitted claims.
- [ ] Add tests for approve, reject, request-changes, and resubmit flows.
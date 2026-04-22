# Task 09 - Approval Workflow and Audit Actions

## Objective

Implement step `6.2` by creating a reusable approval queue and action history for trip expense and compensation workflows.

## Status

Implemented across backend, frontend, and mobile.

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

## Implemented workflow

- The approval workflow is now modeled through generic `approval_requests` and `approval_actions` tables.
- Supported entity types are `TRIP_EXPENSE_CLAIM` and `DRIVER_TRIP_COMPENSATION`.
- Review routing is two-stage: `MANAGER -> ACCOUNTANT`.
- Generic approval-request statuses are `PENDING`, `APPROVED`, `REJECTED`, and `CHANGES_REQUESTED`.
- Recorded approval actions are `SUBMITTED`, `APPROVED`, `REJECTED`, `REQUEST_CHANGES`, and `RESUBMITTED`.
- Source-domain statuses remain the source of truth for expense and compensation records. They are synchronized from the approval request state by the source services instead of duplicating workflow rules in handlers or UI.

## Approval ownership and escalation path

- `MANAGER` performs first-pass review for both supported entity types.
- `ACCOUNTANT` performs the final finance review for both supported entity types.
- `ADMIN` can access the centralized approval queue and read approval history for both entity types.
- Driver mobile remains read-only for approvals: drivers can see approval status, comments, and history, then correct and resubmit from the source record when a reviewer rejects or requests changes.

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

### Implemented backend notes

- Migration `000009_approval_workflow` adds the generic approval tables, source-record foreign keys, indexes, and approval permissions.
- `TripExpenseClaimService` and `DriverTripCompensationService` now delegate approval transitions to `ApprovalService` while preserving source-domain posting behavior.
- Source repositories preload the embedded `approvalRequest` and `approvalRequest.actions` graph so clients can render approval state without extra fan-out calls.
- Central approval APIs are handled by `ApprovalHandler` and wired in `internal/routes/routes.go`.

## Suggested API surface

- `GET /api/approvals/queue`
- `GET /api/approvals/:id`
- `POST /api/approvals/:id/approve`
- `POST /api/approvals/:id/reject`
- `POST /api/approvals/:id/request-changes`
- `POST /api/approvals/:id/resubmit`

All of the above are implemented.

## Frontend implementation

- Add an approval queue workspace for managers and accountants.
- Show source entity details inline so reviewers do not need to navigate away for every decision.
- Show full action history including actor, timestamp, and comment.

### Implemented frontend notes

- A centralized approval workspace is available at `/admin/approvals` for `ADMIN` and `MANAGER`, and at `/accountant/approvals` for `ACCOUNTANT`.
- The workspace renders mixed queues for expense claims and compensation claims.
- Reviewers can inspect source details inline, including expense line items, receipt references, compensation breakdown, and approval action history.
- Reviewer actions exposed in the workspace are approve, reject, and request changes. Accountant compensation approval also supports approved-total input.

## Mobile implementation

- No approval administration on mobile.
- Driver mobile must show current approval status, reviewer comments, and whether resubmission is required.

### Implemented mobile notes

- Driver trip expense and compensation detail screens now render embedded approval status, current stage, full action history, and resubmission-required banners.
- Mobile continues to use the existing source detail and submit endpoints. No separate approval queue or approval actions are exposed on mobile.

## Documentation updates

- Document approval ownership and escalation path.
- Document which roles can approve which entity types.

### Role-to-entity review matrix

| Role | Trip Expense Claim | Driver Trip Compensation |
|------|--------------------|--------------------------|
| `MANAGER` | Stage 1 review | Stage 1 review |
| `ACCOUNTANT` | Final review | Final review |
| `ADMIN` | Queue visibility and approval oversight | Queue visibility and approval oversight |
| `DRIVER` | Read-only source visibility, correction, resubmission | Read-only source visibility, correction, resubmission |

## Acceptance criteria

- Expense and compensation claims can be submitted into a centralized approval queue.
- Every approval decision records actor, timestamp, and comment.
- Rejected or change-requested claims can be corrected and resubmitted.
- Approval status is visible from both source records and the queue.

All acceptance criteria are implemented.

## Validation

- Backend runnable package checks passed:
  - `go test ./internal/models ./internal/database ./internal/repository`
  - `go test ./cmd/api ./internal/routes ./internal/repository ./internal/database`
  - `go test ./internal/approvaltests`
- Frontend approval queue additions passed `npx tsc --noEmit`; the remaining frontend typecheck failure is pre-existing and unrelated in `src/app/api/proxy/route.ts` due duplicate `headers` declarations.
- Mobile approval visibility changes are clean in VS Code diagnostics for the touched Dart files. Full `flutter analyze` could not be executed here because `flutter` is not available on PATH in this environment.

## Implementation checklist

- [x] Create approval migrations and models.
- [x] Add generic approval services and handlers.
- [x] Link expense and compensation domains to approval requests.
- [x] Add FE approval queue.
- [x] Add Mobile status visibility for submitted claims.
- [x] Add tests for approve, reject, request-changes, and resubmit flows.
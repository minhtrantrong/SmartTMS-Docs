# Task 19 Rollout Runbook

## Scope

This rollout closes the hardening phase for Tasks 03-18 with three concrete changes:

- Backend migration `000017_task19_hardening_backfills` backfills missing approval-request links for historical trip expense and driver compensation records, and seeds default fixed-cost allocation rules.
- Frontend admin route guards now resolve dynamic admin paths safely instead of falling back to a broad allow-list.
- Backend, frontend, and mobile regression coverage now includes order-request workflow, backend RBAC middleware, frontend route/workflow helpers, and mobile role-routing/access gating.

## Release Order

1. Take a database snapshot or restore point before applying migrations.
2. Apply backend migrations through `000017_task19_hardening_backfills` on a staging or development-data copy first.
3. Run the Task 19 focused regression checks.
4. Deploy `SmartTMS-BE`.
5. Deploy `SmartTMS-FE` after backend health and auth checks pass.
6. Deploy `SmartTMS-Mobile` only after the final backend contracts are verified against the new build.
7. Leave `SmartTMS-AI` unchanged unless OCR endpoint or secret management changed separately.

## Migration Notes

`000017_task19_hardening_backfills` performs two safe data repairs:

- Seeds four default vehicle allocation rules for `ACTIVE_VEHICLE_COUNT`, `MILEAGE`, `COMPLETED_TRIPS`, and `ACTIVE_DAYS`.
- Reconstructs `approval_requests` and minimal `approval_actions` for historical `trip_expense_claims` and `driver_trip_compensation` rows that were created before approval linking existed.

Rollback note:

- The down migration removes only Task 19 backfill approval records whose actions are fully marked with `[task19 backfill]` comments.
- The down migration removes seeded allocation rules only when they are not already referenced by allocation runs.
- If seeded allocation rules have been used in production allocation runs, prefer restoring from the pre-release database snapshot instead of forcing a migration rollback.

## Focused Regression Commands

### Backend

```powershell
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-BE

# RBAC middleware
go test ./internal/middleware -run "^TestRequireRole_" -count=1

# Order-request workflow slice
go test .\internal\service\order_request_service.go .\internal\service\shipment_service.go .\internal\service\shipment_workflow.go .\internal\service\shipment_journal.go .\internal\service\operation_journal_service.go .\internal\service\order_request_service_test.go -run "^TestOrderRequestService_" -count=1
```

Known caveat:

- Package-wide `go test ./internal/service` is still blocked by the existing compile mismatch in `internal/service/user_service_test.go`.

### Frontend

```powershell
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-FE
npm run test -- src/lib/routePermissions.test.ts src/lib/workflows.test.ts
```

### Mobile

```powershell
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-Mobile
flutter test test/task19_navigation_access_test.dart
```

Known caveat:

- `flutter` must be available on `PATH` for the mobile regression command to run.

## Smoke Checklist

Run these end-to-end checks after deployment:

1. Customer or customer-service creates an order request, reviews it, and converts it to a shipment.
2. Dispatcher assigns a dispatch order and driver acknowledges it.
3. Driver submits a trip expense claim, manager/accountant complete approval, and accountant posts the trip cost.
4. Driver submits trip compensation, reviewers approve it, and finance can see the posted downstream state.
5. Accountant generates a statement, customer-service sends it, customer acknowledges it, and reconciliation can be opened if disputed.
6. Accountant issues an invoice, records a payment, reverses a payment, and confirms invoice collection state updates.
7. Accountant opens fixed-cost allocation rules and confirms the seeded default rules are present.
8. Accountant verifies vehicle revenue and P&L report totals against the underlying invoice and cost ledgers for the same date range.

## Support Handoff Notes

- Customer and accountant mobile sign-ins should land on the unsupported-role screen by design in this release.
- Driver mobile access should still surface trip expenses, driver compensation, and bill scan actions.
- Admin dynamic routes under `/admin/customers/:id` now keep the same allow-list as `/admin/customers` instead of falling back to a wider default.
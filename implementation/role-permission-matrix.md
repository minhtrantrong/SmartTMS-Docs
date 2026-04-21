# Role and Permission Matrix

## Purpose

This document is the canonical reference for the approved SmartTMS role set after Task 01.

## Roles

| Role | Persona | Primary workspace | Notes |
| --- | --- | --- | --- |
| `ADMIN` | System administrator | Admin | Full access across all current domains and future finance domains. |
| `MANAGER` | Operations manager | Admin | Oversees operations, approvals, and reporting with limited finance write access. |
| `DISPATCHER` | Dispatch operations | Dispatcher | Plans routes, assigns capacity, and manages shipment execution. |
| `DRIVER` | Driver | Driver | Handles assigned trips and delivery status updates. |
| `CUSTOMER_SERVICE` | Internal support | Customer Service | Supports customer communication and internal follow-up. No finance write access by default. |
| `CUSTOMER` | External customer | Customer | May only access records belonging to the authenticated customer account. |
| `ACCOUNTANT` | Finance and reporting | Accountant | Owns statement finalization, reconciliation, invoices, payments, costs, and PnL reporting. |

## Permission Groups

### Existing operational permissions

- `USER_READ`, `USER_WRITE`, `USER_DELETE`
- `VEHICLE_READ`, `VEHICLE_WRITE`, `VEHICLE_DELETE`, `VEHICLE_ASSIGN`
- `DRIVER_READ`, `DRIVER_WRITE`, `DRIVER_ASSIGN`
- `SHIPMENT_READ`, `SHIPMENT_WRITE`, `SHIPMENT_DELETE`, `SHIPMENT_APPROVE`, `SHIPMENT_ASSIGN`
- `ROUTE_READ`, `ROUTE_WRITE`, `ROUTE_DELETE`, `ROUTE_OPTIMIZE`
- `CUSTOMER_READ`, `CUSTOMER_WRITE`, `CUSTOMER_DELETE`
- `REPORT_VIEW`, `REPORT_EXPORT`
- `SYSTEM_CONFIG`, `SYSTEM_AUDIT`

### Task 01 finance and approval permissions

- `APPROVAL_READ`, `APPROVAL_WRITE`
- `STATEMENT_READ`, `STATEMENT_WRITE`
- `RECONCILIATION_READ`, `RECONCILIATION_WRITE`
- `INVOICE_READ`, `INVOICE_WRITE`
- `PAYMENT_READ`, `PAYMENT_WRITE`
- `TRIP_COST_READ`, `TRIP_COST_WRITE`
- `FIXED_COST_READ`, `FIXED_COST_WRITE`
- `PNL_VIEW`, `PNL_EXPORT`

### Task 07 trip expense claim permissions

- `TRIP_EXPENSE_CLAIM_READ`
- `TRIP_EXPENSE_CLAIM_WRITE`
- `TRIP_EXPENSE_CLAIM_SUBMIT`
- `TRIP_EXPENSE_CLAIM_REVIEW`

## Role Mapping

| Role | Operational scope | Finance scope | Reporting scope |
| --- | --- | --- | --- |
| `ADMIN` | Full operational access | Full finance access | Full reporting and export |
| `MANAGER` | Full operational oversight | Approval write, finance read | PnL view/export |
| `DISPATCHER` | Shipment, route, driver, vehicle operations plus trip-expense review visibility | None by default | No finance reporting by default |
| `DRIVER` | Assigned trip execution plus own trip-expense claim create, edit, and submit | None by default | None by default |
| `CUSTOMER_SERVICE` | Customer communication and shipment follow-up | Statement/invoice read only | `REPORT_VIEW` only |
| `CUSTOMER` | Own shipments only | Own statements, reconciliations, invoices read only | None by default |
| `ACCOUNTANT` | Customer and shipment read for finance workflows | Full statement, reconciliation, invoice, payment, cost, and trip-expense review ownership | Full reporting and export |

## Task 07 Role Assignment

| Role | Task 07 permissions | Scope |
| --- | --- | --- |
| `ADMIN` | `TRIP_EXPENSE_CLAIM_READ`, `TRIP_EXPENSE_CLAIM_WRITE`, `TRIP_EXPENSE_CLAIM_SUBMIT`, `TRIP_EXPENSE_CLAIM_REVIEW` | Full access across all claims and review actions. |
| `MANAGER` | `TRIP_EXPENSE_CLAIM_READ`, `TRIP_EXPENSE_CLAIM_REVIEW` | Cross-team review and approval oversight. |
| `DISPATCHER` | `TRIP_EXPENSE_CLAIM_READ`, `TRIP_EXPENSE_CLAIM_REVIEW` | Operational review visibility for shipment-linked costs. |
| `DRIVER` | `TRIP_EXPENSE_CLAIM_READ`, `TRIP_EXPENSE_CLAIM_WRITE`, `TRIP_EXPENSE_CLAIM_SUBMIT` | Own claims only; may work in draft and resubmit after rejection. |
| `ACCOUNTANT` | `TRIP_EXPENSE_CLAIM_READ`, `TRIP_EXPENSE_CLAIM_REVIEW` | Finance review, approval, rejection, and posting workflow. |

## Current Implementation Notes

- Backend login and current-user responses return both `role` and resolved permission names.
- Current customer self-scope is resolved by matching `users.email` to `customers.email`.
- This email-based mapping is a temporary bridge until a dedicated user-to-customer relationship is introduced.
- Frontend workspace placeholders exist for `CUSTOMER` and `ACCOUNTANT` so redirects and navigation remain stable while workflow UI is implemented in later tasks.
- Mobile explicitly accepts `CUSTOMER` and `ACCOUNTANT` sessions but routes them to an unsupported-role screen instead of operational dashboards.
- Task 07 is implemented as one trip-expense claim per shipment and driver, with per-item receipt attachments and reviewer workflow states `DRAFT`, `SUBMITTED`, `UNDER_REVIEW`, `APPROVED`, `REJECTED`, and `POSTED`.
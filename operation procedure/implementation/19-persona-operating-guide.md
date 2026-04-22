# Task 19 Persona Operating Guide

## Web Workspaces

### Admin

- Uses `/admin/*` routes for cross-domain oversight.
- Primary queues: approvals, statements, reconciliation, P&L, users, shipments, and audit/journal review.
- Should not be used as the default finance workspace for accountants.

### Manager

- Shares the admin operational workspace for approvals, statements, reconciliation, and P&L oversight.
- First-stage reviewer in the approval workflow for trip expense and driver compensation claims.

### Dispatcher

- Uses `/dispatcher/*` routes for dispatch orders, shipments, vehicles, and operation journal review.
- Does not own accountant-only statements, payments, fixed costs, or P&L pages.

### Customer Service

- Uses `/customer-service/*` routes for order intake, statement delivery support, and reconciliation handling.
- Can access customer admin detail routes that remain mapped to the customer workspace policy.

### Accountant

- Uses `/accountant/*` routes for approvals, statements, invoices, payments, fixed costs, reconciliation, and P&L.
- Owns the final finance review lane for trip expense and driver compensation approvals.
- Should verify that the seeded default allocation rules exist before the first fixed-cost allocation run.

### Customer

- Uses `/customer/*` routes for order visibility, statements, reconciliation, and invoices.
- Does not use the admin workspace.

## Mobile Workspaces

### Driver

- Supported on mobile.
- Primary operational capture flows: dispatch acknowledgment, trip expense submission, driver compensation submission, and bill scan/upload.

### Dispatcher

- Supported on mobile dashboard routing.
- Mobile remains secondary to the web dispatcher workspace for deeper operational administration.

### Customer Service

- Supported on mobile dashboard routing.
- Use web for the full order intake, statements, and reconciliation workflows.

### Admin and Manager

- Supported on the general mobile dashboard.
- Use web for deep finance and approval administration.

### Accountant and Customer

- Intentionally blocked on mobile in this phase.
- Both roles should land on the unsupported-role screen and continue work in the web workspace.

## Support Triage Shortcuts

1. If an admin detail page unexpectedly opens for a driver or customer-service user, verify the pathname is covered by the frontend route-permission matcher.
2. If a historical trip claim or compensation record shows no approval link, confirm migration `000017_task19_hardening_backfills` has been applied.
3. If a fixed-cost workspace is empty on first use, check that the default allocation-rule seeds from Task 19 were created successfully.
4. If mobile role behavior looks wrong, verify the stored role string and confirm `AppRoutes.dashboardForRoleName()` still maps accountant and customer to the unsupported-role screen.
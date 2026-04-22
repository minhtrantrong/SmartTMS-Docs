# Task 02 - Workflow State Machines and Domain Glossary

## Objective

Freeze the business state transitions and domain vocabulary before dependent tickets expand operations, accounting, customer access, and reporting. This document is the canonical reference for lifecycle names, allowed transitions, actors, and ownership boundaries.

## Dependencies

- Task 01

## Scope Status

- Code-enforced now: shipment lifecycle in `SmartTMS-BE`, `SmartTMS-FE`, and `SmartTMS-Mobile`
- Canonicalized here for future implementation: order intake, dispatch order, trip expense, driver compensation, statement, reconciliation, invoice, reporting period
- Rule for all later tasks: do not invent alternate state names in API payloads, UI labels, exports, or reports

## Lifecycle Summary

| Domain | Initial State | Terminal States | Primary Owner |
| --- | --- | --- | --- |
| Shipment | `PENDING` | `DELIVERED`, `CANCELLED` | Operations and Driver |
| Order intake | `SUBMITTED` | `REJECTED`, `CONVERTED_TO_SHIPMENT`, `CANCELLED` | Customer Service |
| Dispatch order | `DRAFT` | `COMPLETED`, `CANCELLED` | Dispatcher |
| Trip expense | `DRAFT` | `POSTED`, `REJECTED` | Driver and Accounting |
| Driver compensation | `DRAFT` | `POSTED`, `REJECTED` | Operations and Accounting |
| Statement | `DRAFT` | `CLOSED` | Customer Service and Accounting |
| Reconciliation | `OPEN` | `CLOSED`, `ESCALATED` | Accounting |
| Invoice | `DRAFT` | `PAID`, `VOID` | Accounting |
| Reporting period | `OPEN` | `CLOSED` | Finance Controller |

## Shipment Lifecycle

### Approved states

- `PENDING`: shipment exists but is not yet fully assigned for execution
- `ASSIGNED`: vehicle and driver commitment exists and the shipment is ready for driver execution
- `PICKED_UP`: cargo has been physically collected
- `IN_TRANSIT`: shipment is actively moving between pickup and delivery
- `DELIVERED`: delivery has completed
- `CANCELLED`: shipment was stopped before completion

### Transition table

| From | To | Actor | Rule |
| --- | --- | --- | --- |
| `PENDING` | `ASSIGNED` | Dispatcher, Manager, Admin | Requires both driver and vehicle assignment |
| `PENDING` | `CANCELLED` | Dispatcher, Manager, Admin | Used when request is abandoned before dispatch |
| `ASSIGNED` | `PICKED_UP` | Driver | Driver-only execution transition |
| `ASSIGNED` | `CANCELLED` | Dispatcher, Manager, Admin | Allowed only before physical pickup |
| `PICKED_UP` | `IN_TRANSIT` | Driver | Driver confirms route execution started |
| `IN_TRANSIT` | `DELIVERED` | Driver | Driver confirms final completion |

### Disallowed shortcuts

- No direct `PENDING -> DELIVERED`
- No direct `ASSIGNED -> DELIVERED`
- No backward transitions from `PICKED_UP`, `IN_TRANSIT`, or `DELIVERED`
- No cancellation after pickup in the current phase of implementation

### Code alignment

- Backend transition guard lives in shipment service, not in handlers
- Invalid shipment transitions return a business-facing `400` message
- FE and Mobile driver actions are derived from the same transition model rather than hard-coded page switches

## Order Intake Lifecycle

### Approved states

- `SUBMITTED`: customer or customer service created the order request
- `UNDER_REVIEW`: request is being checked for serviceability, pricing, and completeness
- `CONFIRMED`: request is approved for conversion planning
- `REJECTED`: request is declined and will not proceed
- `CONVERTED_TO_SHIPMENT`: operational shipment record has been created
- `CANCELLED`: request was withdrawn before conversion

### Transition table

| From | To | Actor | Rule |
| --- | --- | --- | --- |
| `SUBMITTED` | `UNDER_REVIEW` | Customer Service | Intake triage starts |
| `SUBMITTED` | `CANCELLED` | Customer, Customer Service | Withdrawal before review completion |
| `UNDER_REVIEW` | `CONFIRMED` | Customer Service | Commercial and service checks passed |
| `UNDER_REVIEW` | `REJECTED` | Customer Service, Manager | Request cannot be accepted |
| `UNDER_REVIEW` | `CANCELLED` | Customer Service | Intake stopped without confirmation |
| `CONFIRMED` | `CONVERTED_TO_SHIPMENT` | Dispatcher, Customer Service | Shipment creation completed |
| `CONFIRMED` | `CANCELLED` | Customer Service, Manager | Request cancelled before shipment creation |

## Dispatch Order Lifecycle

### Approved states

- `DRAFT`
- `READY_FOR_ASSIGNMENT`
- `ASSIGNED`
- `ACKNOWLEDGED`
- `IN_EXECUTION`
- `COMPLETED`
- `CANCELLED`

### Transition table

| From | To | Actor | Rule |
| --- | --- | --- | --- |
| `DRAFT` | `READY_FOR_ASSIGNMENT` | Dispatcher | Planning data is complete |
| `DRAFT` | `CANCELLED` | Dispatcher | Draft abandoned |
| `READY_FOR_ASSIGNMENT` | `ASSIGNED` | Dispatcher | Driver and vehicle selected |
| `READY_FOR_ASSIGNMENT` | `CANCELLED` | Dispatcher | Assignment halted |
| `ASSIGNED` | `ACKNOWLEDGED` | Driver | Driver confirms receipt |
| `ASSIGNED` | `CANCELLED` | Dispatcher, Manager | Cancel before execution |
| `ACKNOWLEDGED` | `IN_EXECUTION` | Driver | Work starts |
| `IN_EXECUTION` | `COMPLETED` | Driver, Dispatcher | Operational leg complete |

## Trip Expense Lifecycle

### Approved states

- `DRAFT`
- `SUBMITTED`
- `UNDER_REVIEW`
- `APPROVED`
- `REJECTED`
- `POSTED`

### Transition table

| From | To | Actor | Rule |
| --- | --- | --- | --- |
| `DRAFT` | `SUBMITTED` | Driver | Expense claim submitted |
| `SUBMITTED` | `UNDER_REVIEW` | Dispatcher, Accounting | Review opened |
| `UNDER_REVIEW` | `APPROVED` | Accounting | Claim accepted |
| `UNDER_REVIEW` | `REJECTED` | Accounting | Claim denied |
| `APPROVED` | `POSTED` | Accounting | Expense posted to ledger or cost register |

## Driver Compensation Lifecycle

### Approved states

- `DRAFT`
- `SUBMITTED`
- `UNDER_REVIEW`
- `APPROVED`
- `REJECTED`
- `POSTED`

### Transition table

| From | To | Actor | Rule |
| --- | --- | --- | --- |
| `DRAFT` | `SUBMITTED` | Operations | Compensation package prepared |
| `SUBMITTED` | `UNDER_REVIEW` | Accounting | Payroll review starts |
| `UNDER_REVIEW` | `APPROVED` | Accounting, Manager | Amount accepted |
| `UNDER_REVIEW` | `REJECTED` | Accounting, Manager | Amount rejected or returned |
| `APPROVED` | `POSTED` | Accounting | Payroll or payable posting complete |

## Statement Lifecycle

### Approved states

- `DRAFT`
- `GENERATED`
- `SENT`
- `ACKNOWLEDGED`
- `DISPUTED`
- `CLOSED`

### Transition table

| From | To | Actor | Rule |
| --- | --- | --- | --- |
| `DRAFT` | `GENERATED` | Customer Service, Accounting | Statement content finalized |
| `GENERATED` | `SENT` | Customer Service | Delivered to customer contact |
| `SENT` | `ACKNOWLEDGED` | Customer, Customer Service | Customer confirms receipt |
| `SENT` | `DISPUTED` | Customer, Customer Service | Customer challenges content |
| `ACKNOWLEDGED` | `CLOSED` | Customer Service, Accounting | No further action required |
| `DISPUTED` | `CLOSED` | Customer Service, Accounting | Dispute resolved and closed |

## Reconciliation Lifecycle

### Approved states

- `OPEN`
- `UNDER_REVIEW`
- `ADJUSTED`
- `RESOLVED`
- `CLOSED`
- `ESCALATED`

### Transition table

| From | To | Actor | Rule |
| --- | --- | --- | --- |
| `OPEN` | `UNDER_REVIEW` | Accounting | Reconciliation work starts |
| `UNDER_REVIEW` | `ADJUSTED` | Accounting | Adjustments recorded |
| `UNDER_REVIEW` | `RESOLVED` | Accounting | No adjustment needed or issue solved |
| `ADJUSTED` | `RESOLVED` | Accounting | Adjustment accepted |
| `RESOLVED` | `CLOSED` | Accounting | Final close-out complete |
| `UNDER_REVIEW` | `ESCALATED` | Accounting, Manager | Requires commercial or leadership decision |

## Invoice Lifecycle

### Approved states

- `DRAFT`
- `ISSUED`
- `PARTIALLY_PAID`
- `PAID`
- `VOID`

### Transition table

| From | To | Actor | Rule |
| --- | --- | --- | --- |
| `DRAFT` | `ISSUED` | Accounting | Invoice released to customer |
| `ISSUED` | `PARTIALLY_PAID` | Accounting | Payment received but balance remains |
| `ISSUED` | `PAID` | Accounting | Full payment received |
| `PARTIALLY_PAID` | `PAID` | Accounting | Remaining balance settled |
| `DRAFT` | `VOID` | Accounting | Draft cancelled before release |
| `ISSUED` | `VOID` | Accounting, Manager | Only for formal void process |

## Reporting Period Lifecycle

### Approved states

- `OPEN`
- `LOCKED`
- `CLOSED`

### Transition table

| From | To | Actor | Rule |
| --- | --- | --- | --- |
| `OPEN` | `LOCKED` | Finance Controller | Data entry cut-off begins |
| `LOCKED` | `CLOSED` | Finance Controller | Reporting sign-off complete |

### Re-open policy

- Re-open is not a normal transition
- Any reopen must be handled by a future exception workflow, not by mutating `CLOSED` back to `OPEN`

## Domain Glossary

### Customer vs Customer Service

| Term | Meaning | Owner |
| --- | --- | --- |
| Customer | External company or person buying transport service | External account |
| Customer Service | Internal SmartTMS role managing intake, communication, statements, and dispute intake | Internal operations support |

### Order Request vs Shipment vs Dispatch Order

| Term | Meaning | Owner |
| --- | --- | --- |
| Order request | Commercial or service request before operational commitment | Customer Service |
| Shipment | Transport obligation to move goods from pickup to delivery | Operations |
| Dispatch order | Execution instruction for a driver and vehicle against a shipment or shipment leg | Dispatcher |

### Expense Claim vs Cost Posting

| Term | Meaning | Owner |
| --- | --- | --- |
| Expense claim | Submitted operational cost awaiting review | Driver or Operations |
| Cost posting | Approved accounting entry or committed cost record after review | Accounting |

### Statement vs Reconciliation vs Invoice

| Term | Meaning | Owner |
| --- | --- | --- |
| Statement | Customer-facing activity or balance summary | Customer Service and Accounting |
| Reconciliation | Internal matching and discrepancy resolution process | Accounting |
| Invoice | Formal bill requesting payment | Accounting |

### Additional canonical terms

| Term | Meaning | Owner |
| --- | --- | --- |
| Driver compensation | Amount owed to a driver after operational and accounting review | Operations and Accounting |
| Reporting period | Controlled accounting/reporting window for close procedures | Finance Controller |
| Ownership boundary | Team accountable for lifecycle progression, approval, and auditability | Domain-specific |

## Implementation Rules

### Backend

- Keep transition guards in service/domain services, not handlers
- Reject invalid transitions with explicit business messages
- Route all status mutations through one transition path so event logging and approval hooks can attach later
- Require assignment completeness before `ASSIGNED` or later shipment states are persisted

### Frontend

- Build action buttons and editable status options from the canonical workflow maps
- Prefer shared label helpers over page-local status text transforms
- Do not infer permissions from page placement alone; combine role and allowed transition

### Mobile

- Use shared status constants and transition helpers for driver actions
- Do not gate driver actions with screen-local string comparisons when a workflow helper exists
- Keep raw payload values canonical and translate only for display

## Acceptance Status

- [x] Approve state names for each workflow
- [x] Document allowed transitions and actors
- [x] Add backend transition-validation helpers for the implemented shipment lifecycle
- [x] Replace free-form driver-side UI action logic with state-aware action mapping in FE and Mobile
- [x] Publish the domain glossary in `SmartTMS-Docs`
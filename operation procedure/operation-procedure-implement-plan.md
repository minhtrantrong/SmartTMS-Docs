# SmartTMS Operation Procedure Implementation Plan

Date: 2026-04-21

## 1. Purpose

This document captures the implementation plan required to align the current SmartTMS codebase with the operation procedure defined in `operation_procedure.drawio`.

The plan is based on a code-level review of:

- `SmartTMS-BE`
- `SmartTMS-FE`
- `SmartTMS-Mobile`

## 2. Inputs Reviewed

- `SmartTMS-Docs/operation_procedure.drawio`
- `SmartTMS-Docs/MULTI_REPO_DEVELOPMENT_GUIDE.md`
- Current backend routes, models, migrations, repositories, and handlers in `SmartTMS-BE`
- Current web routes, API client, and screen flows in `SmartTMS-FE`
- Current mobile features, models, and API integrations in `SmartTMS-Mobile`

## 3. Executive Summary

The current product implements the core transportation workflow reasonably well:

- internal shipment creation
- shipment assignment
- driver shipment progression
- bill upload
- OCR-assisted document capture
- basic dashboard statistics

However, the current system does not yet implement the full operation procedure.

The largest gaps are:

1. There is no true end-customer lane in the product today.
2. There is no accounting domain for statements, invoices, fixed costs, salary records, or per-vehicle profitability.
3. There is no approval and reconciliation workflow even though parts of the permission model anticipate it.
4. There is no operation journal or audit-style business event log that corresponds to the procedure's control points.

In practical terms, SmartTMS currently supports steps `2`, `5`, and `6` best, supports parts of `1`, `3`, `6.1`, and `10`, and does not yet support the full intent of `4`, `6.2`, `7`, `8`, `9`, `11`, `12`, `13`, and `14`.

## 4. Key Current-State Findings

### 4.1 What is already implemented well

- Shipment CRUD and assignment exist in backend, web, and mobile.
- Driver shipment status updates are implemented.
- Delivered-shipment bill upload and OCR flows are implemented.
- Dashboard reporting exists for fleet and shipment counts.
- Basic cost fields exist on shipments and routes.

### 4.2 Structural gaps

- The system has no `CUSTOMER` role today. It has `CUSTOMER_SERVICE`, which is an internal role, not an external customer persona.
- The system has no `ACCOUNTANT` or equivalent accounting role.
- The backend schema does not contain invoice, statement, fixed-cost, trip-expense, salary, reconciliation, or PnL (Profit and Loss) entities.
- The frontend and mobile clients do not contain dedicated accounting, approval, or customer reconciliation workspaces.

### 4.3 Process gap conclusion

The current product is primarily an operations execution system.

It is not yet a full end-to-end operating system for:

- customer order intake
- customer reconciliation
- trip cost approval
- accounting documents
- invoicing
- per-vehicle financial reporting

## 5. Procedure Coverage Matrix

| Step | Procedure step | Current state | Gap summary |
| --- | --- | --- | --- |
| 1 | KH đặt lệnh | Partial | Shipment creation exists, but only through internal roles; no real customer-facing order placement flow |
| 2 | Tiếp nhận thông tin | Implemented | Dispatcher/admin flows and dashboards support intake and visibility |
| 3 | Báo Cáo hiện trạng xe | Partial | Driver and vehicle status fields exist, but driver self-reporting is not a first-class workflow |
| 4 | Nhật trình điều hành | Missing | No operation journal, dispatch log, or event timeline matching the procedure |
| 5 | Nhận lệnh điều động | Partial | Assignment exists through shipment CRUD, but not as a dedicated dispatch order workflow |
| 6 | Giao hàng, nộp chứng từ | Implemented | Driver status changes, bill upload, and OCR-supported document handling exist |
| 6.1 | Báo chi phí chuyến, lương | Partial | Cost fields exist, but no dedicated trip-expense or salary-reporting workflow |
| 6.2 | Duyệt chi phí, lương | Missing | No approval queue, approval state, or approval APIs/UI |
| 7 | Lập bảng kê, chứng từ | Missing | OCR files exist, but no statement or accounting-document generation workflow |
| 8 | Nhận bảng kê SL và CT | Missing | No customer-facing delivery or acknowledgment flow |
| 9 | Đối chiếu sản lượng | Missing | No reconciliation records, variance handling, or adjustment workflow |
| 10 | Xuất HĐ và ghi nhận DT | Partial | Revenue is inferred from delivered shipment totals, but there is no invoice domain |
| 11 | Doanh thu từng xe | Missing | No vehicle-level revenue aggregation or report |
| 12 | Ghi nhận chi phí, lương | Missing | No trip-cost ledger or salary records |
| 13 | Chi phí cố định | Missing | No fixed-cost entity or allocation workflow |
| 14 | Báo Cáo Kinh doanh PnL (Profit and Loss) từng xe | Missing | No per-vehicle profitability reporting |

## 6. Target Operating Model Decisions

Before implementation starts, these decisions should be made explicitly.

### 6.1 Persona decisions

Decide whether the operation procedure should be implemented with these roles:

- `CUSTOMER`: external customer user/portal access
- `CUSTOMER_SERVICE`: internal service team
- `ACCOUNTANT`: internal accounting/finance operations
- `DISPATCHER`: dispatch and transport operations
- `DRIVER`: operational execution and trip reporting

Recommended decision:

- Add a true `CUSTOMER` role for order placement, statement receipt, and reconciliation.
- Add an `ACCOUNTANT` role for invoice, cost, salary, and PnL (Profit and Loss) workflows.

### 6.2 Workflow ownership decisions

Define who owns each stage:

- order creation and confirmation
- dispatch assignment
- driver status reporting
- trip-expense submission
- expense approval
- statement generation
- reconciliation handling
- invoice issuance
- financial reporting and close

### 6.3 State machine decisions

Formalize business state machines for:

- shipment lifecycle
- dispatch order lifecycle
- trip-expense lifecycle
- reconciliation lifecycle
- invoice lifecycle
- profitability reporting period lifecycle

## 7. Delivery Principles

1. Backend domain and database changes must come first.
2. Finance features should not be implemented by stretching shipment fields further than they already are.
3. Approval and audit capabilities must be first-class, not implicit UI behavior.
4. Mobile should focus on operational capture and driver-side workflows, not general accounting administration.
5. Per-vehicle PnL (Profit and Loss) should only be built after revenue, direct cost, and fixed-cost allocation models are in place.

## 8. Phased Implementation Plan

## Phase 0. Process Alignment and Domain Design

### Objective

Establish the business model and role model needed to implement the procedure cleanly.

### Deliverables

- Approved role model
- Approved workflow/state diagrams
- Approved data ownership for operational vs accounting domains
- Approved terminology and entity definitions

### Backend tasks

- Define missing domain entities and relationships.
- Confirm whether new roles require enum migration updates.
- Define approval, reconciliation, and accounting data models before coding.

### Frontend and Mobile tasks

- Validate which personas need FE, Mobile, or both.
- Confirm whether external customers require a web portal, mobile app, or both.

### Documentation tasks

- Update operation procedure documentation into a versioned process spec.
- Add domain glossary and state transition tables.

### Exit criteria

- No major business ambiguity remains for steps `1`, `6.2`, `7`, `8`, `9`, `10`, `11`, `12`, `13`, and `14`.

## Phase 1. Complete the Operations Workflow

### Objective

Finish the operational part of the flow from order intake through delivery evidence.

### Gaps addressed

- Step `1`
- Step `3`
- Step `4`
- Step `5`

### Backend tasks

- Add a dedicated dispatch/order intake model if shipment is no longer enough to represent the intake stage.
- Add `shipment_events` or `operation_journal` table for business event tracking.
- Add driver self-report endpoints for operational availability and current trip state.
- Add dedicated assignment actions and event logging for dispatch decisions.

### Frontend tasks

- Build a proper order intake workflow for internal staff or external customers.
- Build an operation journal or activity timeline view.
- Add a dispatch order workspace instead of relying only on generic shipment editing.

### Mobile tasks

- Add driver self-report workflow for status such as available, on trip, off duty, on break.
- Add a clearer dispatch-order detail screen for assigned work.

### Suggested data additions

- `operation_journal`
- `shipment_events`
- `driver_status_updates`
- optional `dispatch_orders` if shipment should not represent both order and execution object

### Exit criteria

- A shipment or dispatch activity can be traced through an event history.
- Driver operational status can be reported directly by the driver.
- Intake and assignment are visible as explicit workflow stages.

## Phase 2. Trip Cost and Approval Workflow

### Objective

Implement the post-trip cost and salary submission and approval steps.

### Gaps addressed

- Step `6.1`
- Step `6.2`

### Backend tasks

- Add `trip_expense_claims` table.
- Add `trip_salary_claims` or `driver_trip_compensation` table.
- Add approval state fields and approval history.
- Add approval endpoints and RBAC enforcement.
- Add attachment support if receipts are required.

### Frontend tasks

- Build dispatcher or manager approval queue.
- Build trip-expense and salary review screens.
- Show approval history and rejection reasons.

### Mobile tasks

- Add driver trip-expense submission workflow.
- Add supporting evidence upload for trip expenses.
- Show approval status and rejection feedback to drivers.

### Suggested data additions

- `trip_expense_claims`
- `trip_expense_items`
- `driver_trip_compensation`
- `approval_actions`

### Exit criteria

- Drivers can submit trip cost and salary-related claims.
- Dispatch or management can approve, reject, and comment.
- All approval actions are auditable.

## Phase 3. Customer Statements and Reconciliation

### Objective

Implement the customer-facing document and reconciliation loop.

### Gaps addressed

- Step `7`
- Step `8`
- Step `9`

### Backend tasks

- Add statement or manifest entities for quantity and document packages.
- Add customer acknowledgment and reconciliation records.
- Add variance, dispute, and adjustment structures.
- Add endpoints for sending, viewing, acknowledging, and adjusting statements.

### Frontend tasks

- Build statement generation and review screens for internal staff.
- Build customer-facing or customer-service-assisted statement delivery flow.
- Build reconciliation work queue and adjustment handling.

### Mobile tasks

- Only implement customer-facing mobile features if customer mobile access is a confirmed product decision.
- Otherwise keep this phase web-first.

### Suggested data additions

- `shipment_statements`
- `statement_items`
- `customer_reconciliations`
- `reconciliation_adjustments`

### Exit criteria

- Customers or internal customer-service staff can receive a shipment statement package.
- Quantities and documents can be acknowledged, disputed, and adjusted.
- Reconciliation status is tracked explicitly.

## Phase 4. Accounting Domain and Invoicing

### Objective

Implement invoice issuance, revenue recognition, and cost recording properly.

### Gaps addressed

- Step `10`
- Step `12`
- Step `13`

### Backend tasks

- Add invoice and invoice-line domain.
- Add payment and collection status domain if needed.
- Add trip cost posting domain.
- Add fixed-cost ledger domain.
- Add allocation rules to connect fixed costs to vehicles or periods.

### Frontend tasks

- Build accounting screens for:
  - invoice generation
  - invoice review
  - payment tracking
  - trip cost posting
  - fixed-cost entry
  - cost allocation management

### Mobile tasks

- Limit mobile to operational inputs that feed finance, such as trip claims and proof documents.

### Suggested data additions

- `invoices`
- `invoice_lines`
- `payments`
- `trip_cost_postings`
- `fixed_costs`
- `cost_allocations`

### Exit criteria

- Revenue is recorded through invoices rather than inferred only from delivered shipments.
- Trip costs and fixed costs are persisted as independent accounting records.
- Finance users can manage the accounting workflow without editing shipment data directly.

## Phase 5. Vehicle Revenue and PnL (Profit and Loss) Reporting

### Objective

Implement vehicle-level financial reporting based on authoritative accounting and operational data.

### Gaps addressed

- Step `11`
- Step `14`

### Backend tasks

- Add vehicle revenue aggregation endpoints.
- Add vehicle direct-cost aggregation.
- Add fixed-cost allocation results by vehicle and period.
- Add PnL (Profit and Loss) reporting endpoints or reporting views.

### Frontend tasks

- Build vehicle revenue report.
- Build vehicle PnL (Profit and Loss) report.
- Add period filters, export options, and drill-down views.

### Mobile tasks

- No major new mobile work required beyond supplying upstream data.

### Suggested outputs

- vehicle revenue by period
- vehicle direct costs by period
- fixed-cost allocations by vehicle
- vehicle gross margin
- vehicle net contribution / PnL (Profit and Loss)

### Exit criteria

- Users can view per-vehicle revenue.
- Users can view per-vehicle PnL (Profit and Loss) with drill-down support.
- Figures reconcile back to invoice and cost ledgers.

## Phase 6. Hardening, Controls, and Rollout

### Objective

Make the new workflow reliable, secure, auditable, and maintainable.

### Cross-cutting tasks

- Add audit logging for all business-critical actions.
- Add migration-safe backfills and seed updates.
- Add role-based access tests.
- Add API integration tests for workflow state transitions.
- Add FE and Mobile regression coverage for critical paths.
- Add data export and reporting validation checks.
- Update user documentation and runbooks.

### Exit criteria

- Workflow steps are traceable end-to-end.
- Approval actions are auditable.
- Financial reports reconcile to source records.
- Critical flows have automated regression coverage.

## 9. Repo-by-Repo Work Breakdown

### 9.1 SmartTMS-BE

Primary responsibilities:

- role expansion and RBAC enforcement
- workflow state transitions
- operational journal and audit history
- trip-expense and salary domain
- statement and reconciliation domain
- invoice and accounting domain
- vehicle revenue and PnL (Profit and Loss) reporting APIs

Priority work items:

1. Add missing domain tables and migrations.
2. Add service-layer orchestration for approval and reconciliation workflows.
3. Expose explicit APIs for approvals, statements, invoices, and reports.
4. Add audit/event logging.
5. Add tests for business rules and transitions.

### 9.2 SmartTMS-FE

Primary responsibilities:

- dispatcher workspaces
- manager approval workspaces
- customer-service and/or customer portal flows
- accounting and reporting workspaces

Priority work items:

1. Replace generic shipment-only UI with workflow-specific screens where needed.
2. Add approval queue UX.
3. Add statement and reconciliation screens.
4. Add invoice, cost, fixed-cost, and PnL (Profit and Loss) screens.
5. Add role-aware navigation for new personas.

### 9.3 SmartTMS-Mobile

Primary responsibilities:

- driver operational reporting
- trip-expense capture
- proof/document capture
- limited visibility into approval outcomes

Priority work items:

1. Add driver self-reporting for operational status.
2. Add trip-expense and evidence submission.
3. Keep document capture and OCR integrated with trip workflows.
4. Avoid moving accounting administration into mobile unless there is a confirmed business need.

### 9.4 SmartTMS-AI

Primary responsibilities:

- continue supporting OCR extraction for bill and document evidence

Expected change impact:

- low compared with BE, FE, and Mobile
- may need expanded OCR payload structure only if accounting documents require richer extraction metadata

### 9.5 SmartTMS-Docs

Primary responsibilities:

- process specification
- state diagrams
- role matrix
- implementation backlog and milestones
- acceptance criteria per step

## 10. Recommended Implementation Sequence

Recommended sequence:

1. Phase 0: process and domain alignment
2. Phase 1: close operational workflow gaps
3. Phase 2: implement trip cost and approval workflow
4. Phase 3: implement customer statements and reconciliation
5. Phase 4: implement accounting domain and invoicing
6. Phase 5: implement vehicle revenue and PnL (Profit and Loss) reporting
7. Phase 6: harden, audit, test, and document

Rationale:

- The current system already has enough operational foundation to justify finishing operations first.
- Approval, reconciliation, and accounting should not be layered on top of incomplete operational tracking.
- Vehicle-level PnL (Profit and Loss) is the last stage because it depends on reliable revenue and cost records.

## 11. Recommended Backlog Structure

Create epics in this order:

### Epic A. Role and Workflow Alignment

- Introduce missing roles
- Freeze workflow states
- Approve domain vocabulary

### Epic B. Operations Completion

- Order intake
- Driver status reporting
- Dispatch order workflow
- Operation journal

### Epic C. Trip Cost Approval

- Trip-expense submission
- Salary/trip compensation submission
- Approval queue and audit history

### Epic D. Customer Statement and Reconciliation

- Statement generation
- Acknowledgment flow
- Dispute and adjustment flow

### Epic E. Accounting Core

- Invoices
- Trip cost postings
- Fixed-cost management
- Payment tracking

### Epic F. Vehicle Financial Reports

- Revenue by vehicle
- Cost by vehicle
- PnL (Profit and Loss) by vehicle

### Epic G. Hardening

- tests
- audit logs
- exports
- operational documentation

## 12. Acceptance Criteria by Business Area

### Operations acceptance criteria

- A dispatch-related activity can be traced from intake to delivery.
- Drivers can report operational state directly.
- Delivery evidence is attached and visible in workflow context.

### Approval acceptance criteria

- Every submitted cost item has a status.
- Every approval decision captures actor, timestamp, and comment.
- Rejected items can be corrected and resubmitted.

### Customer reconciliation acceptance criteria

- Customers or customer-service staff can review shipment statements.
- Disputes and adjustments are tracked to resolution.

### Accounting acceptance criteria

- Invoices are generated from approved operational and reconciliation outputs.
- Revenue is recognized from accounting records rather than shipment heuristics alone.
- Costs are recorded independently from shipment master data.

### Reporting acceptance criteria

- Vehicle revenue and vehicle PnL (Profit and Loss) reports can be filtered by period.
- Financial totals reconcile back to source invoices and cost postings.

## 13. Immediate Next Actions

The recommended immediate next actions are:

1. Approve whether SmartTMS will support a real `CUSTOMER` role and an `ACCOUNTANT` role.
2. Approve the target state machines for shipment, expense approval, reconciliation, and invoice flows.
3. Start backend schema design for:
   - operation journal
   - trip-expense and salary claims
   - statement and reconciliation records
   - invoice and cost ledgers
4. Create implementation tickets grouped by the epics listed in this document.
5. Begin with Phase 0 and Phase 1 before building finance reports.

## 14. Summary

SmartTMS is already strong enough to support core transport operations, but it does not yet fulfill the full operation procedure.

To close the gap, the product must evolve from an operations-first shipment system into a broader workflow platform that includes:

- explicit customer or customer-service intake and reconciliation flows
- explicit approval workflows
- explicit accounting entities and workspaces
- explicit vehicle-level financial reporting

The recommended path is to deliver this in phases, starting with role and workflow alignment, then finishing operational control points, then implementing approval, customer reconciliation, accounting, and finally per-vehicle profitability reporting.
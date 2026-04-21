# SmartTMS Accounting Feature Implementation Plan

## 1. Overview
The Accounting module will serve as the financial core of the Smart Transport Management System (SmartTMS). It will consolidate all financial data, tracking revenue from shipment bills, and calculating operational expenses such as vehicle repairs, fuel (oil), toll fees, and employee salaries. This plan outlines the database updates, backend APIs, frontend UI, and AI integrations necessary to implement this feature.

## 2. Database Schema Updates (SmartTMS-BE/migrations)
We will create new tables to track specific financial outflows, integrating them with the existing database schema (e.g., `users`, `vehicles`, `shipment_bills`).

### New Tables:
1. **`expenses`**
   - **Columns**: `id` (UUID), `vehicle_id` (UUID, nullable), `driver_id` (UUID, nullable), `type` (ENUM: 'FUEL', 'TOLL', 'REPAIR', 'MAINTENANCE', 'OTHER'), `amount` (DECIMAL), `currency` (VARCHAR), `date_incurred` (TIMESTAMP), `description` (TEXT), `receipt_url` (VARCHAR), `created_at`, `updated_at`.
   - **Purpose**: Log daily operational costs.

2. **`salaries`**
   - **Columns**: `id` (UUID), `user_id` (UUID - references `users`), `base_salary` (DECIMAL), `bonus` (DECIMAL), `deductions` (DECIMAL), `net_pay` (DECIMAL), `payment_date` (TIMESTAMP), `period_start` (DATE), `period_end` (DATE), `status` (ENUM: 'PENDING', 'PAID').
   - **Purpose**: Track payroll and employee compensation.

3. **`ledger_transactions` (Optional/View)**
   - To unify revenue (from `shipment_bills.total_cost`) and expenses (from `expenses` and `salaries`) for high-level reporting and profit/loss (P&L) statements.

## 3. Backend Implementation (SmartTMS-BE)

### Architecture Layers:
*   **Models (`internal/models/`)**: Create `expense.go` and `salary.go` matching the database schema.
*   **DTOs (`internal/dto/`)**: Create payload structures for creating expenses, running payroll, and filtering financial reports (e.g., `AccountingDashboardSummaryDTO`).
*   **Repositories (`internal/repository/`)**: 
    *   `ExpenseRepository`: CRUD for expenses.
    *   `SalaryRepository`: CRUD for payroll.
    *   `AccountingRepository`: Complex aggregation queries (e.g., "Total revenue vs total expense grouped by month", "Total repair costs for Vehicle X").
*   **Services (`internal/service/`)**:
    *   `AccountingService`: Orchestrates fetching shipment bill totals, validating salaries, and computing P&L logic.
*   **Handlers (`internal/handler/`)**:
    *   `AccountingHandler`: Maps HTTP requests to the `AccountingService`.

### Proposed API Endpoints:
*   `GET /api/v1/accounting/summary` - Aggregated Data (Revenue, Expense, Profit) for dashboard.
*   `GET /api/v1/accounting/expenses` - List expenses (with filters for date range, vehicle, type).
*   `POST /api/v1/accounting/expenses` - Record a new expense.
*   `GET /api/v1/accounting/salaries` - List payroll records.
*   `POST /api/v1/accounting/salaries` - Generate/record a salary payment.

## 4. Frontend Implementation (SmartTMS-FE)

### API Integration:
*   Update `src/lib/api.ts` with new accounting service endpoints.

### User Interface (Dashboard & Pages):
*   **`/accounting` (Main Dashboard)**:
    *   Visual charts (e.g., Recharts) showing Monthly Revenue vs. Expenses.
    *   Key Performance Indicators (KPIs): Total Profit, Top Expenses (Fuel vs. Repairs).
*   **`/accounting/expenses`**:
    *   A comprehensive data table to view, filter, and export expenses.
    *   "Add Expense" modal forms with fields for category, amount, vehicle assignment, and receipt upload.
*   **`/accounting/payroll`**:
    *   View employee salaries, trigger payments, and view historical payroll data.
*   **Role-Based Access Control (RBAC)**: Ensure that only users with `ADMIN` and `MANAGER` roles have view/edit permissions for the accounting routes.

## 5. AI/OCR Integration (SmartTMS-AI)
Leverage the existing AI service to automate financial data entry:
*   **Enhance OCR API**: Extend the `SmartTMS-AI` to support generic receipt parsing (toll tickets, fuel receipts, maintenance invoices).
*   **Endpoint**: `POST /api/ocr/receipt` to extract fields like `amount`, `date`, `category`, and `merchant`.
*   **Workflow**: When a driver or manager uploads a photo of a toll/fuel receipt in the Frontend, invoke the OCR service to auto-fill the "Add Expense" form.

## 6. Execution Steps
1.  **Draft Migrations**: Create SQL up/down migrations for `expenses` and `salaries` tables.
2.  **Backend CRUD**: Implement models, generic repository methods, services, and handlers for Expenses and Salaries.
3.  **Aggregation Engine**: Write complex SQL counts/sums in `AccountingRepository` for the Dashboard features.
4.  **Frontend Core Pages**: Build the basic lists and forms for Expenses and Payroll.
5.  **Frontend Dashboard**: Implement data visualization charts.
6.  **AI OCR Extension**: Train/prompt the OCR model to recognize receipts and wire it to the frontend upload component.

## 7. Advanced Considerations & Enterprise Features
To make the accounting feature truly robust and enterprise-ready for a Transport Management System, consider the following:

### 7.1 Invoicing & Accounts Receivable/Payable
*   **Customer Invoicing:** Generating PDF invoices directly from `shipment_bills` and sending them to clients.
*   **Payment Status Tracking:** Tracking which shipment bills are `UNPAID`, `PARTIALLY_PAID`, or `PAID`, as well as overdue alerts.
*   **Vendor Management:** Tracking accounts payable for external mechanics, fuel stations, or third-party logistics partners.

### 7.2 Trip/Driver Settlements (Advances & Reimbursements)
*   **Cash Advances:** Drivers are often given direct cash or card advances before a trip for tolls and fuel.
*   **Reconciliation:** Comparing the trip advance against the actual expenses submitted (via OCR receipts) to calculate the final settlement owed to or from the driver.

### 7.3 Asset Depreciation & Maintenance Ledger
*   **Vehicle Depreciation:** Trucks and vans are heavily depreciating assets. Calculating monthly/yearly depreciation is critical for an accurate Profit & Loss (P&L) statement.
*   **Maintenance Amortization:** Large repair costs (like an engine rebuild) might need to be amortized over several months rather than hitting the P&L in a single month.

### 7.4 Taxes and Multi-Currency
*   **Tax Management:** Handling output tax (e.g., VAT charged on shipments) vs. input tax (VAT paid on fuel/repairs) for tax reporting.
*   **Multi-Currency:** If your fleets cross borders, you will need exchange rate tables to convert foreign toll/fuel expenses into the base currency.

### 7.5 Compliance & Security Workflow (Maker/Checker)
*   **Expense Approval Workflows:** A driver submits an expense -> a `MANAGER` reviews it -> an `ADMIN` approves payout.
*   **Immutable Audit Logs:** Financial records should not be easily hard-deleted. Use soft deletes or an append-only ledger system to ensure auditability.

### 7.6 Bank Integration / Payment Gateways
*   **Automated Payouts:** APIs (like Stripe, PayPal, or regional bank APIs) to automate salary disbursements and expense reimbursements.
*   **Bank Reconciliation:** Providing a way to check software ledger balances against uploaded CSV bank statements.

## 8. Transport-Specific Financial KPIs & Flow
Because this is a logistics/transport business, generic accounting is not enough. The system should track specific industry metrics:

### 8.1 Unit Analytics (Cost per Mile/Km)
*   **Revenue per Mile (RPM):** Total shipment revenue divided by total distance driven.
*   **Cost per Mile (CPM):** Total operational expenses (fuel, driver pay, truck maintenance) divided by total distance.
*   **Empty Miles / Deadhead Tracking:** Recording and factoring in the cost of driving returning trips without cargo (generating $0 revenue).

### 8.2 Complex Driver Pay Models
*   Instead of standard salaries, many drivers operate on:
    *   **Per-Mile Pay:** Paid varying rates based on loaded vs. empty miles.
    *   **Percentage of Load:** Driver receives a fixed percentage (e.g., 20%) of the total `shipment_bills` value.
    *   **Owner-Operator Settlements:** Deducting chargebacks (e.g., if the company pays for the contractor's cargo insurance or fuel, it must be subtracted from their gross pay).

### 8.3 Freight Surcharges & Accessorial Charges
*   **Fuel Surcharge (FSC):** Automatically adjusting customer billing based on the fluctuating national average cost of fuel to protect profit margins.
*   **Detention and Demurrage:** Billing customers an hourly rate if the driver is kept waiting excessively long at the pickup or drop-off facility.
*   **Lumper Fees:** Billing for third-party labor hired to unload a trailer at a warehouse.

### 8.4 Regional Fleet Tax Reporting (e.g., IFTA)
*   If applicable, tracking the fuel purchased and miles driven in each specific state/province/region to apportion fuel taxes correctly (International Fuel Tax Agreement or regional equivalent).
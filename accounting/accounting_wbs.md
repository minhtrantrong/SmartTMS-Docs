# SmartTMS Accounting Feature - Implementation-Ready WBS

This document converts the accounting plan into repo-aligned implementation tasks for SmartTMS. It is intentionally more specific than `accounting_plan.md` and reflects the current codebase structure:

- Backend currently uses `uint64` / `bigint` style IDs, not UUIDs.
- Backend routes currently live under `/api/...`, not `/api/v1/...`.
- Frontend feature pages currently live under `/admin/...`, `/driver/...`, and `/dispatcher/...`.
- Frontend pages currently consume `src/lib/apiProxy.ts` rather than `src/lib/api.ts`.
- AI OCR currently exposes `/apis/ocr/extract` and the backend already proxies uploaded files to that service.
- Current `shipment_bills` stores OCR payloads for uploaded documents; it is not yet the source of truth for bill totals or recognized revenue.

The goal is to deliver a usable v1 accounting module first, then move advanced enterprise features into a later phase.

---

## Phase 0: Lock Implementation Decisions Before Writing Code

### 0.1 Freeze the v1 architecture contract
- [ ] Confirm that v1 backend routes will use `/api/accounting/...` so they match the current router style in `SmartTMS-BE/internal/routes/routes.go`.
  - Do not introduce `/api/v1` only for accounting unless the entire backend is being versioned at the same time.
  - Expected v1 route family:
    - `GET /api/accounting/summary`
    - `GET /api/accounting/expenses`
    - `POST /api/accounting/expenses`
    - `PUT /api/accounting/expenses/:id`
    - `DELETE /api/accounting/expenses/:id`
    - `POST /api/accounting/expenses/parse-receipt`
    - `GET /api/accounting/salaries`
    - `POST /api/accounting/salaries`
    - `PUT /api/accounting/salaries/:id` if payroll edits are allowed in v1
  - Done when the route list is agreed and reused consistently in backend handlers, Swagger docs, frontend API client, and UI links.

### 0.2 Freeze the frontend route strategy
- [ ] Confirm that v1 frontend screens will live under the existing admin route tree.
  - Recommended paths:
    - `SmartTMS-FE/src/app/admin/accounting/page.tsx`
    - `SmartTMS-FE/src/app/admin/accounting/expenses/page.tsx`
    - `SmartTMS-FE/src/app/admin/accounting/payroll/page.tsx`
  - Do not implement top-level `/accounting` pages unless the existing admin navigation model is being changed.
  - Update `src/components/auth/ProtectedRoute.tsx` and `src/components/layout/Sidebar.tsx` together so the new routes are both protected and visible.
  - Done when navigation, route guards, and page locations all follow the same admin convention.

### 0.3 Freeze ID and foreign-key strategy
- [ ] Decide whether accounting tables follow the current database ID style or introduce UUIDs.
  - Recommended for v1: keep `bigserial` primary keys and `bigint` foreign keys, because the current backend models use `uint64` IDs across `users`, `drivers`, `vehicles`, `routes`, `shipments`, and `shipment_bills`.
  - If UUIDs are still required, treat that as a schema-wide design change and explicitly update model field types, repository signatures, route param parsing, and frontend types before implementing accounting features.
  - Done when the selected ID strategy is reflected in migrations, GORM models, DTOs, handler param parsing, and TypeScript types.

### 0.4 Freeze the revenue source of truth for dashboard v1
- [ ] Decide what counts as revenue in the first accounting dashboard.
  - Important: the current `shipment_bills` table does not hold financial totals; it stores OCR data for uploaded bill images.
  - Recommended for v1: use `shipments.total_cost` as the revenue source.
  - Decide whether revenue recognition is:
    - delivered shipments only, or
    - paid shipments only using `shipments.payment_status`.
  - Recommended for v1 operational dashboard: use delivered shipments for `recognizedRevenue`, and optionally expose a separate `paidRevenue` if finance wants cash-basis visibility.
  - Done when summary calculations, API docs, UI labels, and test cases all reference the same rule.

### 0.5 Freeze the OCR integration flow
- [ ] Confirm that the browser will not call the AI OCR service directly.
  - The AI service requires an API key; that key should remain server-side.
  - Recommended flow:
    - Frontend uploads receipt image to backend.
    - Backend forwards file to AI service.
    - AI service returns parsed receipt fields.
    - Backend normalizes the result and returns it to the frontend form.
  - This keeps the API secret out of the client and reuses the existing backend-to-AI integration pattern already used in `bill_handler.go`.
  - Done when the receipt parse endpoint exists only on the backend public API, while AI remains an internal dependency.

### 0.6 Freeze v1 scope and explicitly defer later items
- [ ] Lock the v1 scope so implementation does not drift into the advanced ideas section too early.
  - Include in v1:
    - expense storage and management
    - payroll record storage and management
    - accounting summary dashboard
    - OCR-assisted expense entry
    - basic charts and filtering
  - Defer to later phases:
    - invoices and accounts receivable workflows
    - customer payment follow-up and overdue logic
    - driver cash advances and settlements
    - asset depreciation and repair amortization
    - multi-currency conversion
    - tax reporting
    - maker/checker approval workflow
    - bank reconciliation
    - transport KPI engine such as RPM and CPM
  - Done when the v1 backlog only contains work needed to ship the first usable accounting release.

---

## Phase 1: Database Design and Migrations

### 1.1 Define final accounting table specifications
- [ ] Write the final SQL-level specification for `expenses` and `salaries` before creating migrations.
  - `expenses` recommended columns for v1:
    - `id bigserial primary key`
    - `vehicle_id bigint null references vehicles(id) on delete set null`
    - `driver_id bigint null references drivers(id) on delete set null`
    - `type expense_type not null`
    - `amount numeric(12,2) not null check (amount > 0)`
    - `currency varchar(3) not null`
    - `date_incurred timestamptz not null`
    - `description text null`
    - `receipt_url varchar(500) null`
    - `created_at timestamptz not null default current_timestamp`
    - `updated_at timestamptz not null default current_timestamp`
  - `salaries` recommended columns for v1:
    - `id bigserial primary key`
    - `user_id bigint not null references users(id) on delete restrict`
    - `base_salary numeric(12,2) not null default 0`
    - `bonus numeric(12,2) not null default 0`
    - `deductions numeric(12,2) not null default 0`
    - `net_pay numeric(12,2) not null`
    - `payment_date timestamptz null`
    - `period_start date not null`
    - `period_end date not null`
    - `status salary_status not null default 'PENDING'`
    - `created_at timestamptz not null default current_timestamp`
    - `updated_at timestamptz not null default current_timestamp`
  - Decide whether to add optional accounting-only columns now:
    - `created_by` / `updated_by` for audit visibility
    - `notes` on salary records
    - `receipt_ocr_data jsonb` if parsed OCR payload must be retained
  - Done when SQL column types and constraints are final enough to implement without guessing.

### 1.2 Decide enum and validation strategy
- [ ] Define database enums or check constraints for accounting-specific statuses.
  - `expense_type` recommended values:
    - `FUEL`
    - `TOLL`
    - `REPAIR`
    - `MAINTENANCE`
    - `OTHER`
  - `salary_status` recommended values:
    - `PENDING`
    - `PAID`
  - Prefer explicit database enums if you want parity with other strongly typed status fields.
  - Prefer `varchar + check` only if migration simplicity is more important than enum reuse.
  - Done when enum values match backend constants, frontend select options, OCR category mapping, and Swagger examples.

### 1.3 Create the core accounting migration
- [ ] Create `SmartTMS-BE/migrations/000007_add_accounting_core.up.sql`.
  - Include enum creation if enums are used.
  - Create the `expenses` table.
  - Create the `salaries` table.
  - Add indexes needed for list screens and summary queries:
    - `expenses(date_incurred)`
    - `expenses(type)`
    - `expenses(vehicle_id)`
    - `expenses(driver_id)`
    - `salaries(user_id)`
    - `salaries(status)`
    - `salaries(period_start, period_end)`
  - Add a unique constraint if payroll duplication must be prevented for the same user and period.
    - Recommended candidate: `(user_id, period_start, period_end)`.
  - If soft deletion is required for finance auditability, add `deleted_at`; otherwise document that v1 follows the current hard-delete convention and plan a later audit hardening phase.
  - Done when the migration is idempotent at the schema level and matches the final specification.

### 1.4 Create the rollback migration
- [ ] Create `SmartTMS-BE/migrations/000007_add_accounting_core.down.sql`.
  - Drop dependent indexes if they were created separately.
  - Drop `salaries` and `expenses` tables in dependency-safe order.
  - Drop custom enums only after all dependent objects are removed.
  - Done when the database can move up and down one version cleanly without leaving broken types or partial tables.

### 1.5 Add optional reporting view migration
- [ ] Create `SmartTMS-BE/migrations/000008_add_accounting_reporting_view.up.sql` only after the core tables are stable.
  - Recommended purpose: simplify monthly dashboard queries and provide a unified ledger-like data source.
  - Recommended first version of `ledger_transactions`:
    - revenue rows derived from `shipments.total_cost`
    - expense rows derived from `expenses.amount`
    - payroll rows derived from `salaries.net_pay`
  - Use explicit sign conventions.
    - revenue positive
    - expenses and salaries negative
  - Add a corresponding `000008_add_accounting_reporting_view.down.sql` to drop the view.
  - Done when the view definition is simple, documented, and not hiding business rules that should live in service code.

### 1.6 Validate migrations against the local dev database
- [ ] Run migration verification using the repo-standard workflow.
  - Commands:
    - `make migrate-up`
    - `make migrate-down`
    - `make migrate-up`
  - If Make variables are not set, use the documented direct command:
    - `migrate -path migrations -database "postgres://smarttms_user:smarttms_dev_pass@localhost:5432/smarttms_dev?sslmode=disable" up`
  - Inspect resulting schema with `psql` queries against `information_schema.columns` and `pg_indexes`.
  - Done when columns, constraints, defaults, indexes, and rollback all behave as expected.

---

## Phase 2: Backend Implementation in SmartTMS-BE

### 2.1 Create accounting models
- [ ] Add `SmartTMS-BE/internal/models/expense.go`.
  - Define `ExpenseType` constants.
  - Define `Expense` struct with GORM tags matching the new migration.
  - Add optional relations for `Vehicle` and `Driver` preloading.
  - Include `TableName()` returning `expenses`.
  - Use JSON field names that match frontend expectations such as `vehicleId`, `driverId`, `dateIncurred`, and `receiptUrl`.
  - Done when the model can be used in repository queries without extra mapping hacks.

- [ ] Add `SmartTMS-BE/internal/models/salary.go`.
  - Define `SalaryStatus` constants.
  - Define `Salary` struct with relation to `User`.
  - Include `TableName()` returning `salaries`.
  - Done when the model captures all persisted payroll fields and uses consistent JSON naming.

- [ ] Update existing model files only where cross-entity relations are needed.
  - Add reverse relations only if they will actually be preloaded or serialized.
  - Avoid bloating unrelated models if the accounting API will load related entities explicitly.
  - Done when model dependencies remain readable and no circular serialization problems are introduced.

### 2.2 Create accounting DTOs
- [ ] Add `SmartTMS-BE/internal/dto/accounting.go`.
  - Create request DTOs:
    - `ExpenseListQuery`
    - `CreateExpenseRequest`
    - `UpdateExpenseRequest`
    - `ParseReceiptResponse`
    - `SalaryListQuery`
    - `CreateSalaryRequest`
    - `UpdateSalaryRequest` if editing is allowed
    - `AccountingSummaryQuery`
  - Create response DTOs:
    - `ExpenseResponse`
    - `SalaryResponse`
    - `AccountingSummaryResponse`
    - `MonthlyAccountingPoint`
    - `ExpenseCategoryBreakdown`
  - Reuse `PaginationRequest`, `PaginationResponse`, and `ListResponse` patterns already present in `internal/dto/dto.go`.
  - Include binding tags for all expected filters and payloads.
  - Done when handlers can bind query strings and JSON bodies without custom manual parsing except file uploads.

### 2.3 Define repository contracts
- [ ] Add `SmartTMS-BE/internal/repository/expense_repository.go`.
  - Implement interface methods such as:
    - `Create(expense *models.Expense) error`
    - `FindByID(id uint64) (*models.Expense, error)`
    - `List(filter ExpenseFilter) ([]models.Expense, int64, error)`
    - `Update(expense *models.Expense) error`
    - `Delete(id uint64) error`
  - Add filter support for:
    - date range
    - type
    - vehicle
    - driver
    - pagination
    - sort field and order
  - Preload related `Vehicle`, `Driver`, and `Driver.User` data only where the UI needs names and labels.
  - Done when the repository can support both the expense page and dashboard calculations without N+1 queries.

- [ ] Add `SmartTMS-BE/internal/repository/salary_repository.go`.
  - Implement create, get, list, update, and optional delete methods.
  - Support filters for user, role if needed, payment status, period range, and payment date range.
  - If payroll duplication should be blocked, add `FindByUserAndPeriod` or equivalent helper.
  - Done when salary listing and salary creation flows have all required repository support.

- [ ] Add `SmartTMS-BE/internal/repository/accounting_repository.go`.
  - Implement aggregation methods needed by the dashboard.
  - Recommended methods:
    - `GetRevenueTotals(filter SummaryFilter)`
    - `GetExpenseTotals(filter SummaryFilter)`
    - `GetSalaryTotals(filter SummaryFilter)`
    - `GetMonthlyRevenueVsExpense(filter SummaryFilter)`
    - `GetExpenseBreakdownByType(filter SummaryFilter)`
    - `GetTopExpenseVehicles(filter SummaryFilter)` if vehicle ranking is part of the dashboard
  - Use SQL that is explicit about date boundaries and revenue recognition rules.
  - Prefer `date_trunc('month', ...)` for monthly charts.
  - Done when one summary endpoint can be built without loading every row into application memory.

### 2.4 Extract reusable OCR client logic from bill handling
- [ ] Refactor the existing backend-to-AI OCR request flow into a reusable service or helper.
  - Current OCR forwarding logic lives inside `internal/handler/bill_handler.go`.
  - Move the shared multipart request logic into a reusable location such as:
    - `SmartTMS-BE/internal/service/ocr_service.go`, or
    - `SmartTMS-BE/internal/service/ai_ocr_service.go`
  - Keep support for:
    - multipart file forwarding
    - `X-API-Key` header injection
    - timeout handling from config
    - normalized error handling
  - Update bill handling to use the shared OCR client so accounting receipt parsing does not duplicate HTTP glue code.
  - Done when both bill OCR and accounting receipt OCR use the same backend integration path.

### 2.5 Implement accounting services
- [ ] Add `SmartTMS-BE/internal/service/expense_service.go`.
  - Validate that `amount > 0`.
  - Validate `currency` format and length.
  - Validate referenced `vehicle_id` and `driver_id` if provided.
  - Normalize `description` and reject invalid enum values.
  - Decide how `receipt_url` is populated in v1:
    - nullable field only
    - local file path
    - external storage URL
  - If delete is allowed, enforce role restrictions at service or handler layer consistently.
  - Done when expense creation and editing rules are centralized outside the handler.

- [ ] Add `SmartTMS-BE/internal/service/salary_service.go`.
  - Calculate `net_pay = base_salary + bonus - deductions`.
  - Reject negative `net_pay` unless business rules explicitly allow it.
  - Validate `period_start <= period_end`.
  - Validate eligible payroll users.
    - Decide whether payroll covers only `DRIVER` users or all employee roles.
  - Block duplicate payroll records for the same period if that rule is enabled.
  - When status changes to `PAID`, decide whether `payment_date` becomes required and auto-set if missing.
  - Done when payroll creation is deterministic and does not depend on UI calculations.

- [ ] Add `SmartTMS-BE/internal/service/accounting_service.go`.
  - Combine repository totals into a single summary DTO.
  - Recommended summary fields:
    - `recognizedRevenue`
    - `paidRevenue` if included
    - `operationalExpenses`
    - `salaryExpenses`
    - `totalExpenses`
    - `netProfit`
    - `profitMargin`
    - `monthlyRevenueVsExpense`
    - `expensesByType`
    - `topExpenseVehicles` if included
  - Ensure divide-by-zero safe profit margin logic.
  - Keep business formulas in the service, not the handler.
  - Done when a single service call can populate the accounting dashboard page.

### 2.6 Implement accounting handlers
- [ ] Add `SmartTMS-BE/internal/handler/accounting_handler.go`.
  - Recommended public methods:
    - `GetSummary`
    - `GetExpenses`
    - `CreateExpense`
    - `UpdateExpense`
    - `DeleteExpense`
    - `ParseExpenseReceipt`
    - `GetSalaries`
    - `CreateSalary`
    - `UpdateSalary` if salary edits are allowed
  - Use the existing handler style:
    - constructor injection
    - `gin.Context`
    - consistent JSON error responses
    - Swagger annotations above each handler method
  - Use `ShouldBindQuery` for filters and `ShouldBindJSON` for payloads.
  - For file uploads, use `c.FormFile("file")` and call the shared OCR client.
  - Done when the entire accounting API is surfaced through one or more handlers that match current backend conventions.

### 2.7 Register routes and middleware
- [ ] Update `SmartTMS-BE/internal/routes/routes.go`.
  - Initialize the new repositories, services, and handler(s).
  - Add `accounting := api.Group("/accounting")`.
  - Recommended access rules:
    - summary, salary list, salary create: `ADMIN`, `MANAGER`
    - expense list: `ADMIN`, `MANAGER`
    - expense create: `ADMIN`, `MANAGER`, and optionally `DRIVER` if self-submitted expenses are required in v1
    - expense update and delete: `ADMIN`, `MANAGER`
    - receipt parse: same roles as expense create
  - If drivers can submit expenses, add ownership validation instead of exposing all expenses to all drivers.
  - Done when RBAC is explicit and matches frontend route visibility.

### 2.8 Update backend configuration and env handling
- [ ] Extend backend config only if a dedicated receipt parse URL is introduced on the AI service.
  - Add `AI_OCR_RECEIPT_URL` if backend should call a separate AI endpoint from the existing text extraction endpoint.
  - Keep `AI_API_SECRET_KEY` server-side only.
  - Update any example env docs or setup instructions if new config keys are added.
  - Done when local setup for backend and AI remains reproducible.

### 2.9 Add Swagger and backend docs
- [ ] Add Swagger annotations for all new accounting endpoints.
  - Include request body schemas, multipart upload docs, query params, and error responses.
  - Regenerate docs using `make swagger`.
  - Ensure new routes appear in `SmartTMS-BE/docs/swagger.yaml` and `docs.go`.
  - Done when the API can be explored and tested from Swagger without undocumented payloads.

---

## Phase 3: AI and OCR Implementation in SmartTMS-AI

### 3.1 Preserve the existing low-level OCR endpoint
- [ ] Keep `POST /apis/ocr/extract` working as the generic text extraction endpoint.
  - Do not break current consumers while adding accounting receipt parsing.
  - Keep API-key middleware on the router.
  - Done when existing bill OCR behavior still works after accounting changes.

### 3.2 Add structured receipt parsing endpoint
- [ ] Add a structured endpoint such as `POST /apis/ocr/receipt`.
  - Target files:
    - `SmartTMS-AI/apis/ocr_api.py`
    - `SmartTMS-AI/controllers/ocr_controller.py`
    - `SmartTMS-AI/models/ocr_model.py`
    - `SmartTMS-AI/routers/ocr_router.py` if route export changes are needed
  - Response contract should include at minimum:
    - `amount`
    - `currency`
    - `date`
    - `category`
    - `merchantName`
    - `rawText`
    - `warnings` or `confidence` so low-certainty fields can be manually reviewed
  - Map parsed category values to backend accounting enums.
  - Done when the AI service returns a normalized structure the backend can pass directly to the expense form.

### 3.3 Implement receipt parsing logic
- [ ] Build receipt parsing on top of OCR text extraction.
  - Use OCR text as the first stage.
  - Add parsing logic for common receipt patterns:
    - fuel receipt totals
    - toll ticket totals
    - maintenance invoice totals
  - Normalize dates to ISO-compatible strings.
  - Normalize currency to ISO 3-letter code if possible.
  - Return `warnings` for missing fields rather than fabricating values.
  - Done when test samples with partial or noisy OCR still return safe, reviewable output.

### 3.4 Add AI-side validation and file handling rules
- [ ] Validate receipt upload file types and size limits.
  - Reuse the existing supported extensions list where possible.
  - Ensure malformed PDFs and unreadable images return clear 4xx or 5xx errors.
  - Do not silently swallow parsing failures; include actionable error details.
  - Done when backend callers can distinguish bad input from transient OCR failures.

### 3.5 Verify end-to-end OCR behavior
- [ ] Test the AI receipt endpoint with sample fuel, toll, and repair documents.
  - Save sample inputs and expected outputs under a repo-appropriate docs or temp location if needed.
  - Verify that backend can forward multipart files and parse the response.
  - Done when at least one happy-path sample works for each core expense category.

---

## Phase 4: Frontend Implementation in SmartTMS-FE

### 4.1 Create accounting TypeScript types
- [ ] Update `SmartTMS-FE/src/types/index.ts`.
  - Add types for:
    - `Expense`
    - `Salary`
    - `ExpenseType`
    - `SalaryStatus`
    - `AccountingSummary`
    - `MonthlyAccountingPoint`
    - `ExpenseBreakdown`
    - `ParseReceiptResult`
    - any filter/query helper types used by the pages
  - Keep field names aligned with backend JSON responses.
  - Done when new UI pages can be built without `any`.

### 4.2 Extend the active API client used by pages
- [ ] Update `SmartTMS-FE/src/lib/apiProxy.ts` first, because the current feature pages import that client.
  - Add `accountingAPI` methods such as:
    - `getSummary(params)`
    - `getExpenses(params)`
    - `createExpense(data)`
    - `updateExpense(id, data)`
    - `deleteExpense(id)`
    - `parseReceipt(formData)`
    - `getSalaries(params)`
    - `createSalary(data)`
    - `updateSalary(id, data)` if editing is enabled
  - Keep request formatting consistent with existing pagination and auth behavior.
  - If direct-mode API support in `src/lib/api.ts` still matters, mirror the same methods there or document that `apiProxy.ts` is now the canonical client.
  - Done when pages can call accounting APIs without bypassing the existing auth and proxy behavior.

### 4.3 Add protected accounting routes
- [ ] Create the admin accounting route tree.
  - Files to add:
    - `SmartTMS-FE/src/app/admin/accounting/page.tsx`
    - `SmartTMS-FE/src/app/admin/accounting/expenses/page.tsx`
    - `SmartTMS-FE/src/app/admin/accounting/payroll/page.tsx`
  - Optionally add `src/app/admin/accounting/layout.tsx` if shared filter controls, tabs, or breadcrumbs are needed.
  - Update `ROUTE_PERMISSIONS` in `src/components/auth/ProtectedRoute.tsx`.
    - Recommended roles: `ADMIN`, `MANAGER`
  - Update `src/components/layout/Sidebar.tsx` to add an Accounting navigation item.
  - Done when authorized users can navigate to accounting screens from the existing admin shell.

### 4.4 Create reusable accounting UI components
- [ ] Add a dedicated accounting component folder if page files start growing too large.
  - Recommended location: `SmartTMS-FE/src/components/accounting/`
  - Recommended components:
    - `AccountingSummaryCards.tsx`
    - `RevenueExpenseChart.tsx`
    - `ExpenseBreakdownChart.tsx`
    - `ExpenseTable.tsx`
    - `ExpenseFormModal.tsx`
    - `ReceiptUploadField.tsx`
    - `PayrollTable.tsx`
    - `PayrollFormModal.tsx`
  - Reuse existing UI building blocks from `src/components/ui/`.
  - Done when the accounting pages stay readable and feature logic is not buried in one massive page component.

### 4.5 Build the accounting summary dashboard page
- [ ] Implement `src/app/admin/accounting/page.tsx`.
  - Include filters for date range and optional quick presets such as current month and last 90 days.
  - Show KPI cards for:
    - recognized revenue
    - paid revenue if exposed
    - operational expenses
    - salary expenses
    - total expenses
    - net profit
    - profit margin
  - Use `recharts` for:
    - monthly revenue vs expense chart
    - expense category breakdown chart
  - Add loading, error, and empty states.
  - Format currency and percentages consistently.
  - Done when managers can open one page and understand the business position for the selected date range.

### 4.6 Build the expenses management page
- [ ] Implement `src/app/admin/accounting/expenses/page.tsx`.
  - Fetch paginated data from backend using query-string filters.
  - Include filters for:
    - date range
    - type
    - vehicle
    - driver
    - search text if description or merchant search is supported
  - Table columns should include at least:
    - date incurred
    - type
    - amount
    - currency
    - vehicle
    - driver
    - description
    - receipt indicator
    - created at
  - Add create and edit flows if backend supports update.
  - Add delete flow only if finance policy allows it in v1.
  - Done when finance users can list, filter, create, and maintain expense records from one screen.

### 4.7 Build the expense form and OCR-assisted upload flow
- [ ] Implement the expense create/edit modal or drawer.
  - Use `react-hook-form` because it is already installed in the frontend dependencies.
  - Required form fields for v1:
    - type
    - amount
    - currency
    - date incurred
    - description
    - optional vehicle
    - optional driver
  - Add receipt upload flow:
    - user selects image or PDF
    - frontend sends multipart request to backend parse endpoint
    - show loading state while parsing
    - populate form fields with parsed values
    - mark parsed values as editable so user can correct them
    - display warnings if OCR confidence is low or fields are missing
  - Do not auto-submit expenses from OCR; parsed values must still be reviewed.
  - Done when a user can upload a receipt, review suggested values, and submit a valid expense without leaving the form.

### 4.8 Decide and implement expense export behavior
- [ ] Clarify what "export expenses" means in v1.
  - If export should include all filtered rows across pages, add a backend CSV export endpoint.
  - If export can be limited to current filtered client-side data, clearly label that behavior.
  - Recommended server-side route if needed later:
    - `GET /api/accounting/expenses/export?from=...&to=...&type=...`
  - Done when export behavior is explicit and does not silently omit non-visible records.

### 4.9 Build the payroll page
- [ ] Implement `src/app/admin/accounting/payroll/page.tsx`.
  - Show paginated payroll records with filters for:
    - employee
    - status
    - period range
    - payment date range
  - Show columns for:
    - employee name
    - role
    - base salary
    - bonus
    - deductions
    - net pay
    - period start
    - period end
    - payment status
    - payment date
  - Add create salary record flow.
  - If editing is allowed, prevent accidental mutation of already paid records unless policy allows it.
  - Done when payroll records can be created and reviewed without backend-only tooling.

### 4.10 Implement payroll form rules
- [ ] Create a salary entry form that matches backend validation.
  - Inputs:
    - employee selector
    - base salary
    - bonus
    - deductions
    - period start
    - period end
    - status
    - payment date when status is `PAID`
  - UI validations:
    - positive or zero monetary fields
    - end date cannot be before start date
    - warn if a likely duplicate period exists
  - Use backend-calculated `net_pay` as source of truth even if frontend shows a live preview.
  - Done when the frontend and backend cannot disagree on salary calculations.

### 4.11 Add empty states, loading states, and permissions feedback
- [ ] Ensure the accounting module follows the same UX quality bar as the rest of the admin screens.
  - Add skeleton or spinner states for charts and tables.
  - Add retry affordances on API failure.
  - Show empty states when no data exists for the selected range.
  - Keep access control behavior consistent with `ProtectedRoute`.
  - Done when the module is usable even on first-run datasets and slow network conditions.

---

## Phase 5: Testing, Verification, and Release Readiness

### 5.1 Backend automated tests
- [ ] Add tests for accounting services and repositories.
  - Test expense creation validation.
  - Test salary net-pay calculation.
  - Test duplicate payroll prevention if enabled.
  - Test summary aggregation formulas.
  - Test date-range filters and monthly grouping.
  - Done when critical accounting formulas are covered by automated tests rather than only manual checking.

### 5.2 Backend route and RBAC tests
- [ ] Add API-level verification for the accounting endpoints.
  - Verify `ADMIN` and `MANAGER` access to summary and payroll endpoints.
  - Verify unauthorized users receive 401 or 403 as appropriate.
  - If drivers can create expenses, verify they cannot list or edit unrelated records.
  - Done when role-based access is proven instead of assumed.

### 5.3 AI parsing verification
- [ ] Add repeatable validation for receipt parsing quality.
  - Maintain a small sample set of test receipts.
  - Verify parsed amount, category, and date against expected values.
  - Document known weak cases such as blurry images or handwritten repair notes.
  - Done when OCR regressions can be noticed before release.

### 5.4 Frontend verification
- [ ] Run the existing frontend quality gates.
  - `npm run lint`
  - `npm run build`
  - Verify all new admin accounting pages compile and route correctly.
  - Important: the current frontend does not appear to have an automated UI test runner configured.
  - Decide whether to:
    - add a test stack now, or
    - rely on lint/build plus manual QA for v1.
  - Done when the chosen verification approach is explicit and completed.

### 5.5 End-to-end manual QA scenario
- [ ] Execute one end-to-end flow from receipt upload to dashboard visibility.
  - Suggested manual scenario:
    - create or choose a vehicle and driver
    - upload a fuel receipt
    - parse receipt values through backend and AI
    - submit expense
    - confirm expense appears in filtered list
    - create salary record
    - refresh dashboard
    - confirm totals and charts update correctly
  - Done when the full accounting loop works across BE, FE, AI, and database layers.

### 5.6 Documentation and operational readiness
- [ ] Update docs to reflect the shipped module.
  - Update Swagger docs and regenerate backend docs.
  - Add a short usage guide for the accounting screens and OCR flow.
  - Document any required env vars and service dependencies.
  - If revenue source is based on `shipments.total_cost`, document that clearly so users do not confuse it with OCR bill uploads.
  - Done when a new developer can understand and run the accounting stack without reverse-engineering the code.

---

## Phase 6: Post-v1 Expansion Backlog

These items should remain out of the core v1 delivery unless explicitly prioritized.

### 6.1 Customer invoicing and accounts receivable
- [ ] Add invoice records instead of treating OCR shipment bills as financial invoices.
- [ ] Track invoice payment status, due dates, partial payments, and overdue balances.
- [ ] Generate PDFs and customer-facing invoice exports.

### 6.2 Driver advances and settlement workflows
- [ ] Model driver cash advances before trips.
- [ ] Reconcile advances against submitted expenses.
- [ ] Show outstanding amounts owed to or from the driver.

### 6.3 Asset depreciation and repair amortization
- [ ] Add depreciation schedules for vehicles.
- [ ] Split major repairs across amortization periods when required.
- [ ] Include these costs in profit reporting without distorting one-time monthly spikes.

### 6.4 Multi-currency and tax support
- [ ] Add exchange-rate tables and base-currency normalization.
- [ ] Separate tax-inclusive and tax-exclusive amounts.
- [ ] Build reporting for VAT or regional transport taxes where relevant.

### 6.5 Approval workflows and audit hardening
- [ ] Add maker/checker approval states for expenses and payroll.
- [ ] Replace hard deletes with soft deletes or immutable audit trails.
- [ ] Record who created, reviewed, approved, and paid each financial record.

### 6.6 Banking and reconciliation
- [ ] Import bank statements.
- [ ] Match software records to bank transactions.
- [ ] Highlight unreconciled payouts and missing receipts.

### 6.7 Transport-specific KPI engine
- [ ] Compute revenue per mile or kilometer.
- [ ] Compute cost per mile or kilometer.
- [ ] Add empty-mile tracking and owner-operator settlement logic.
- [ ] Add surcharges such as fuel surcharge, detention, demurrage, and lumper fees.

---

## Suggested Delivery Order

Use this sequence to reduce integration risk and avoid building UI on unstable contracts.

1. Lock the Phase 0 decisions.
2. Ship migration `000007_add_accounting_core`.
3. Implement backend models, DTOs, repositories, and services.
4. Extract shared backend OCR client logic from bill handling.
5. Ship backend accounting endpoints and Swagger docs.
6. Add AI structured receipt parsing endpoint.
7. Add frontend types and `accountingAPI` client methods.
8. Build admin accounting pages and sidebar/RBAC changes.
9. Validate the full OCR-to-expense-to-dashboard flow.
10. Move deferred enterprise features into follow-up work only after v1 is stable.
# SmartTMS Operation Procedure Implementation Tasks

This folder breaks the operation procedure implementation plan into execution-ready tasks.

Use these files as the implementation backlog for `SmartTMS-BE`, `SmartTMS-FE`, `SmartTMS-Mobile`, `SmartTMS-AI`, and `SmartTMS-Docs`.

## Delivery rules

- Database migrations and backend contracts come before frontend and mobile screens.
- Do not extend `shipments` into a catch-all accounting model. Add dedicated finance tables instead.
- All workflow actions that change business state must produce an auditable event.
- Mobile is for operational capture first. Avoid pushing accounting administration into mobile.
- Vehicle revenue and PnL reporting start only after invoice, direct-cost, and fixed-cost domains exist.

## Suggested execution sequence

| ID | Task | Depends on |
| --- | --- | --- |
| 01 | Role model and permission matrix | - |
| 02 | Workflow state machines and domain glossary | 01 |
| 03 | Order intake and customer access | 01, 02 |
| 04 | Operation journal and shipment events | 02 |
| 05 | Driver operational status self-reporting | 02, 04 |
| 06 | Dispatch order workflow | 02, 03, 04 |
| 07 | Trip expense claims | 01, 02, 04 |
| 08 | Driver trip compensation and salary claims | 01, 02, 04 |
| 09 | Approval workflow and audit actions | 07, 08 |
| 10 | Shipment statements and document packages | 04, 06 |
| 11 | Statement delivery and customer acknowledgment | 01, 03, 10 |
| 12 | Reconciliation disputes and adjustments | 09, 10, 11 |
| 13 | Invoice domain and generation | 01, 10, 12 |
| 14 | Trip cost postings and direct-cost ledger | 09 |
| 15 | Fixed-cost ledger and allocation rules | 01, 02 |
| 16 | Payment tracking and collection state | 13 |
| 17 | Vehicle revenue reporting | 12, 13 |
| 18 | Vehicle PnL reporting | 14, 15, 17 |
| 19 | Hardening, backfills, tests, and rollout | 03-18 |

## File conventions

- Each task file includes objective, scope, deliverables, implementation details, and acceptance criteria.
- File names are numbered so the folder can be executed in sequence.
- Cross-repo changes should follow the current local development guide in `SmartTMS-Docs/MULTI_REPO_DEVELOPMENT_GUIDE.md`.

## Implementation notes

- Backend role and enum changes must update both `internal/models/models.go` and the SQL migrations in `SmartTMS-BE/migrations`.
- Frontend role changes must update `SmartTMS-FE/src/types/index.ts`, auth handling, and role-aware routing.
- Mobile role or API changes must update `SmartTMS-Mobile/lib/core/network/api_client.dart` and related feature screens.
- If any new backend route or DTO is added, refresh Swagger output in `SmartTMS-BE/docs` and then align FE and Mobile clients.
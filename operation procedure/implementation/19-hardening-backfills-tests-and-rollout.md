# Task 19 - Hardening, Backfills, Tests, and Rollout

## Objective

Implement phase `6` so the new workflow is safe to release, auditable, and maintainable.

## Dependencies

- Task 03 through Task 18

## Deliverables

- Audit and business-event coverage review.
- Migration-safe backfills and seed updates.
- Automated regression coverage across backend, frontend, and mobile.
- Updated runbooks, rollout plan, and support documentation.

## Backend implementation

- Review every business-critical action and confirm it writes either an approval action, operation-journal entry, or both.
- Add backfill migrations or scripts for any data required by the new domains, for example:
  - seeding new permissions and roles
  - initializing report defaults
  - linking historical shipments to generated statement or event records where feasible
- Add automated tests for:
  - role-based access control
  - workflow transition rules
  - approval and reconciliation rules
  - reporting totals against source data
- Regenerate Swagger docs after all route and DTO work is complete.

## Frontend implementation

- Add regression coverage for the highest-risk workflows:
  - order intake and conversion
  - dispatch assignment and acknowledgment
  - trip claim submission and approval
  - statement delivery and reconciliation
  - invoice issuance and payment updates
  - revenue and PnL report filters
- Verify route guards and role-aware navigation for every supported role.

## Mobile implementation

- Add regression coverage for:
  - driver operational status reporting
  - dispatch acknowledgment
  - trip expense submission
  - compensation submission
  - bill or proof upload interactions that feed later finance steps
- Confirm unsupported roles are handled safely in auth and navigation.

## Documentation updates

- Update `SmartTMS-Docs/MULTI_REPO_DEVELOPMENT_GUIDE.md` if startup order, environment variables, or service contracts change.
- Add rollout runbooks for migration order, feature flags if used, and rollback notes.
- Add user-facing operating guides for new personas and new workspaces.

## Rollout checklist

- [ ] Migrations are ordered and tested on a copy of development data.
- [ ] Seed updates create at least one test account per new role.
- [ ] Swagger docs are regenerated.
- [ ] FE and Mobile clients are aligned to final backend contracts.
- [ ] Reporting totals are reconciled against source ledgers before release.
- [ ] Critical workflows have smoke-test scripts.

## Acceptance criteria

- New workflows are traceable end to end.
- RBAC is enforced across backend and UI layers.
- Financial reports reconcile to source records.
- Documentation is current enough for local setup, QA, and support handoff.

## Implementation checklist

- [ ] Complete audit coverage review.
- [ ] Add migration backfills and seed updates.
- [ ] Add backend workflow and RBAC tests.
- [ ] Add FE regression coverage for high-risk flows.
- [ ] Add Mobile regression coverage for operational capture flows.
- [ ] Refresh docs and rollout runbooks.
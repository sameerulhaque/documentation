# Testing (single source)

Replaces the former UAT playbooks, test specification and logs (originals in git history).

## Safety rules
See `docs/guides/GORETAILOS_AGENT_GUIDE.md` §1. In short: synthetic shops only, deleted afterwards.

## What to test (master checklist)
Auth (sign-in, sign-up, reset, Google, SSO) · onboarding · **tenant isolation (P0)** · role
permissions per role · dashboard · catalog/products/variants · **POS (P0)**: sessions, cart,
discounts, payments, credit limits, receipts, returns, cash in/out, close · offline POS and sync ·
inventory, transfers, counts · purchasing and receiving · accounting postings · storefront cart,
checkout, account, SEO · HR, payroll, CRM, loyalty, restaurant, reports · licensing.

## Seeder
The UAT data seeder (admin area, owner-only) creates demo data idempotently with "UAT" names;
posted/settled transactions are done manually. Danger-zone reset is owner-only and only for UAT
shops.

## Automated tests in the code
Commands and settings: root `README.md` → "Tests". Database-backed ERP checks: `scripts/erp/synthetic-*.test-run.ts`.

## Status
Most routes passed UAT; outstanding items are tracked in root `roadmap.md`.

## Playwright test-execution lessons (was root `GORETAILOS_TEST_CASES_EXECUTION*.MD`)
Those two files were one-off prompts used to build `tests/e2e/` and the test specification; the
work is done. Lessons kept:
- Code and migrations are authoritative over older docs (roles, permissions, grants).
- Use two QA shops (A and B) to prove tenant isolation; personas per role (owner, admin, manager,
  cashier, accountant, inventory, staff) plus an unauthenticated visitor and a platform admin.
- Passwords only from environment (`E2E_*` / `PW_*`), never in the repo; saved sessions in
  `tests/.auth/` (gitignored); one global setup creating `storageState` per persona.
- Use absolute `baseURL` (`PW_BASE_URL`), dismiss the first-run Quick Tour by seeding localStorage,
  prefer semantic selectors (roles/labels), account for modal dialog focus behaviour.
- Every test case records: area, priority, roles, preconditions, data, route, steps, expected UI
  and data results, side effects, negative assertions and cleanup.
- Review while testing for: functional, tenancy, authorization, security, financial and inventory
  correctness; fix only small, clearly-scoped defects.

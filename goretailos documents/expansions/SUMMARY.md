# Expansions (single source)

How each large area was planned and built. Originals in git history.

| Area | Delivered |
| --- | --- |
| HR & Payroll | Employee documents & compliance, lifecycle actions (probation, transfer, promotion, salary review), attendance with correction requests, versioned documents, payroll runs. Modules `employees`, `payroll`. Qatar-specific compliance was planned only. |
| CRM & Loyalty | Activities/tasks, customer metrics and segments (orders excluding cancelled), reward-claim lifecycle around the existing points ledger, governance and an outbox for external call-centre (VOXODial) integration. |
| Advanced Procurement | Requisition → approval → RFQ → supplier quotations and comparison → award → normal Purchasing; supplier qualification/risk; external vendor portal with hashed, expiring invite tokens. |
| Restaurant | Deterministic analytics, floors/tables, kitchen tickets and display, waitlist, modifiers, ingredient reservations. |
| BI / Reporting | Read-only, role- and module-aware KPI and report suites for sales, inventory, purchasing, accounting; executive dashboard; KPI builder. |
| SaaS layer | Plans, module licences, quotas, platform admin (enforcement off in production). |
| Promotions | Gift-with-purchase on the online store. |
| Manufacturing | Bills of materials, versions, theoretical costing, production planning. |
| Security audits | Least-privilege Data API grants and production migration reconciliation; rules now in `docs/supabase-migration/DATA_API_GRANT_CONVENTION.md`. |

Code wins over old notes: migrations once marked "not applied" are in `supabase/migrations/` and
were applied by the owner; the SaaS layer once called "missing" exists
(`src/lib/licensing.server.ts`, `/platform`).

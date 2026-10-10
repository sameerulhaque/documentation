# Integration — GoRetailOS × GoOS Identity × GoCoreOS (single source)

Replaces all former integration documents (full originals are in git history). Machine-readable
contracts stay in `contracts/` because GoCoreOS validates against them. Where old documents
disagreed, the code wins.

> Step-by-step setup with every screen: [`GOCOREOS_INTEGRATION_GUIDE.md`](GOCOREOS_INTEGRATION_GUIDE.md).
> Building a new app (e.g. GoLogisticOS) on this ERP: [`NEW_GOOS_APP_ON_GORETAILOS_ERP.md`](NEW_GOOS_APP_ON_GORETAILOS_ERP.md).

## 1. Roles
- **GoRetailOS = provider and source of truth.** Business logic, data and rules stay here.
- **GoCoreOS = host.** Owns its shell/navigation, a server-side proxy and a module loader; mounts
  GoRetailOS screens as remote ES modules (no iframe, no rebuilt screens).
- **GoOS Identity** (`go-oss-passport.lovable.app`, central Supabase Pro project) = shared sign-in
  and consent app. Every Go*OS app keeps its own users, roles, RLS and data.

## 2. Single sign-on (R7) — proven end to end
- Code `src/lib/sso/`; routes `/api/sso/start`, `/api/sso/callback`, `/sso/complete`; switch
  `SSO_ENABLED`; issuer from `SSO_ISSUER_URL` (legacy fallback kept).
- Flow: OIDC + PKCE → ID token validated (issuer, audience, ES256 JWKS, Basic client auth) →
  identity matched by **(issuer, subject), never email** → one-time local-session bridge
  (generateLink + verifyOtp).
- SSO never grants roles; local `shop_users` roles stay authoritative. Email/password and Google
  sign-in stay available. Users link/unlink in ERP Connections.
- Server-only tables: `sso_auth_requests`, `sso_identities`, `sso_login_handoffs`.
- Setup steps: `GOCOREOS_INTEGRATION_GUIDE.md` Stage A–B; new apps: `NEW_GOOS_APP_ON_GORETAILOS_ERP.md`.

## 3. ERP API (R3–R6)
- One splat route `src/routes/api/erp/v1/$.ts` → `src/lib/erp/erp.server.ts`; kill switch
  `ERP_API_ENABLED` (404 when off). No CORS — GoCoreOS calls it server-side through its proxy.
- Auth: client credentials (shown once to a platform admin) → short-lived ES256 tokens (JWKS,
  rotation, revocation, audit, rate limits).
- Org↔shop **link**: GoCoreOS `POST /links` with `requested_scopes` → GoRetailOS admin approves
  → GoCoreOS confirms. The shop is always derived from the verified link, never from input.
  A link only receives scopes it requested AND the client is allowed.
- Async operations: idempotency keys, ordering, retry/backoff, dead-letter, signed callbacks
  (`GORETAILOS_CALLBACK_SECRET` on the GoCoreOS side), reconciliation, durable worker.
- All ERP tables SERVER_ONLY (service_role grants).
- Host secrets and click-by-click setup: `GOCOREOS_INTEGRATION_GUIDE.md` Stages C–H.

## 4. Scopes
`org:read, operations:read/write, catalog:read/write, pricing:write, inventory:read/write,
warehouses:read, locations:write, purchasing:read/write, suppliers:write, payments:write,
sales:read/write, crm:read/write, accounting:read/write, finance_master:write, reports:read/write`.
High-impact scopes (`accounting:write, inventory:write, payments:write, pricing:write,
sales:write, pos:write`) may be allowed on client/link but are **rejected on service tokens** —
usable only via delegated grants, with `ERP_COMMANDS_ENABLED` on and the per-link business action
ticked.

## 5. Delegated writes
Exact schema: `contracts/DELEGATED_AUTHORIZATION_CONTRACT.md`.
1. Challenge: 2 min, single use, bound to client/link/shop/org/operation/body hash.
2. Fresh GoOS ID token for the signed-in user; audience = GoCoreOS's own GoOS Identity client ID
   (saved in GoRetailOS as the client's "Sign-in app ID" — not the ERP client ID).
3. Grant: 5 min; user must be SSO-linked and hold an active shop role; revocation/demotion
   re-checked.
4. Body hash = SHA-256 of canonical JSON.
Master-data CRUD: 30 stable operation IDs in `contracts/master-data-write-operations-1.0.0.json`.

## 6. Installable packages (seven groups)
Contract `erp-package/3.0` (`contracts/erp-package-3.0.schema.json`), manifests in
`contracts/packages/` (only the newest version matters):
1. Master Data & Catalog (`goretailos.catalog`) — install first; others depend on it
2. Inventory & Purchasing 3. CRM & Loyalty 4. Sales & Orders 5. Accounting & Finance
6. Dashboards & Reports 7. KPI Builder + Executive Dashboard
45 modules in total (registry `src/lib/erp/erp-modules.ts`). POS, HR, Restaurant, Storefront and
Marketing are not ERP packages.

## 7. Shared screens and automatic updates
- One screen, two data sources: the real portal route component is compiled with ERP adapters
  (`src/components/screens/SharedScreenRemote.tsx`), scoped CSS, SRI hashes.
- Release: `scripts/erp/release.ts` (`--check` CI gate, `--rollback <ver>`, `--unpin`); registry
  `public/erp-remotes/release-state.json`; discovery `/capabilities/packages/updates`.
- From 1.16.0 the bundle self-updates, verifies SHA-384, and rolls back on failure (skips the bad
  version 24 h). Coordinated/breaking releases never auto-activate. Opt-out prop `autoUpdate:false`.
  Code `src/lib/erp/remote-self-update.ts`.
- Image upload/delete and manual stock adjustment work through existing services.
- Portal-only by design: payment-proof review, platform KPI admin, barcode lifecycle, accounting
  vouchers/fiscal periods/dimensions/bank-rec tab.
- GoCoreOS import checker must only read real `import`/`from` statements (fixed on their side).

## 8. Rollback
See `GOCOREOS_INTEGRATION_GUIDE.md` "Turning it off"; baseline tag `pre-erp-integration`; SQL in `rollback/`.

## 9. Status (checked in code, 10 Oct 2026)
- Current release **1.16.0**. Switches `ERP_API_ENABLED`, `ERP_COMMANDS_ENABLED`, `SSO_ENABLED`
  are server settings, off by default.
- Open: publishing, live GoCoreOS verification, ERP order creation via the non-POS order service,
  stock reservations (no service exists). See root `roadmap.md`.

## Stock holds and ERP order channel (release 1.17.0)
- Requires `docs/supabase-migration/20261011000001_stock_reservations_and_erp_channel.sql` on the business database. Until applied, hold commands answer "Stock holds are not available yet" and the portal hides the Stock holds panel.
- Commands: `inventory.reservation.create|release|extend`, `inventory.reservations.list` (status required: held/confirmed/released/expired/all), `sales.order.reserve` (hold + saved order details), `sales.order.confirm` (hold → sale), `sales.order.create` now defaults to channel `erp` and accepts `reservation_id`.
- Holds raise `inventory_balances.reserved`; Available = on hand − holds. Confirm runs `record_reserved_sale` (release + `record_pos_sale` in one transaction). With stock enforcement on, POS/storefront sales cannot use stock held by active holds. Expired holds stop counting immediately and are cleaned on the next reserve/list.
- New commands are off per link until enabled in Business actions; delegation and `ERP_COMMANDS_ENABLED` still apply.

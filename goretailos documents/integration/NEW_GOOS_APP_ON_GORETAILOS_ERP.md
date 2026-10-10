# Building a new Go*OS app (e.g. GoLogisticOS) on top of GoRetailOS ERP

**Audience:** Claude / Lovable working in a new project started from GoCoreOS code (first case:
**GoLogisticOS**).

**Private repos — no cross-repo access.** GoRetailOS, GoCoreOS and the new app are separate private
repositories. No project can read another's files, and there are no submodules, shared packages or
remixes across workspaces. So:
- **Starting the code:** the owner duplicates GoCoreOS into a new private repo (git clone + push to a
  new remote, or GitHub "duplicate"), then connects that repo to a new Lovable project. It is a
  one-time snapshot — later GoCoreOS changes are not inherited automatically.
- **Getting the rules in:** the owner manually uploads (or pastes) this file as
  `docs/GORETAILOS_ERP_RULES.md`, plus `GOCOREOS_INTEGRATION_GUIDE.md` and the `contracts/` files,
  into the new project. Do not assume any other GoRetailOS file exists there.
- **At runtime** the apps talk only over HTTPS (ERP API, JWKS, remote module URLs, signed callbacks)
  with the new app's own credentials. Live discovery (`/capabilities/*`, `/erp-remotes/…/manifest.json`)
  is the source of truth; uploaded contract copies are a snapshot and can be stale — if they disagree,
  trust the live endpoint and ask the owner for refreshed copies.
- When something is missing on the GoRetailOS side, write the request as text for the owner to paste
  into the GoRetailOS project (Part 7) — never guess its code.

Then paste Part 6 as the first prompt.

---

## Part 1 — The one rule

> **GoRetailOS owns every ERP record. The new app never keeps its own copy of stock, warehouses,
> products, suppliers, customers, orders or accounts.**

- Anything the user enters that is ERP data (stock moved, goods received, a transfer, an expense,
  a voucher, a customer) is sent to GoRetailOS as a **command**. GoRetailOS runs its existing
  service (stock rules, average cost, journal entries, tax) and the result appears immediately in
  GoRetailOS **and** in every installed module (Inventory, Accounting …) in GoCoreOS and the new app,
  because they all read the same records.
- The new app may keep **its own domain data** (e.g. for logistics: shipments, routes, drivers,
  vehicles, delivery proofs, tracking events) in its own database. Those rows only store
  **references** to GoRetailOS (variant ID, location ID, order ID, transfer ID) — never quantities
  or balances.
- If GoRetailOS has no command for something you need, **stop and ask** for it to be added in
  GoRetailOS. Never write a second stock/accounting engine, never write to GoRetailOS tables directly.

```text
GoLogisticOS (own: shipments, drivers, routes)
   │  reads  ─────────────►  GoRetailOS ERP API  (/api/erp/v1/orgs/{org}/…)
   │  commands (delegated) ►  existing GoRetailOS services → same tables
   │  mounts remote modules ◄ /erp-remotes/<module>/<version>/remote.js (same as GoCoreOS)
   ▼
GoCoreOS, GoRetailOS portal and GoLogisticOS all show the same stock/accounts instantly
```

---

## Part 2 — What the duplicated GoCoreOS code already gives you (reuse, do not rebuild)

| Piece | In GoCoreOS | Change for the new app |
| --- | --- | --- |
| GoOS sign-in (OIDC + PKCE, identity link, local session) | yes | Register a **new** OAuth client on GoOS Identity; new callback URL |
| ERP connector (server-side token, proxy, retries) | `src/lib/goretailos/` | New ERP client ID/secret from GoRetailOS |
| Delegated write flow (challenge → fresh ID token → grant → command) | yes | Sign-in app ID in GoRetailOS = the new app's GoOS client ID |
| Module loader + Marketplace (install, version sync, rollback) | yes | Keep as is; same seven packages |
| Signed status callback endpoint | yes | New callback URL + new `GORETAILOS_CALLBACK_SECRET` |

**Never copy secrets from GoCoreOS.** Every app gets its own GoOS client, ERP client, secrets and
links. Remove GoCoreOS-only features you don't need after duplicating; keep the connector untouched.

---

## Part 3 — One-time setup for the new app
Follow `GOCOREOS_INTEGRATION_GUIDE.md` Part 2, Stages A–H, replacing "GoCoreOS" with the new app
(the owner uploads that guide into the new project). Differences only:
- Stage A: register a **new** GoOS client for the new app; its callback is on the new domain.
- Stage C: client name = new app; tick only the scopes it needs; sign-in app ID = new GoOS client ID.
- Stage D: the same secret names, with the new app's values.

---

## Part 4 — Which GoRetailOS endpoints to use

Base: `https://goretailos.com/api/erp/v1`. Always called **from the server**, never the browser
(no CORS). The shop is taken from the link; never send a shop ID.

### Discover first (never hard-code)
- `GET /capabilities/modules` — modules, screens, versions
- `GET /capabilities/write-operations` — master-data write operation IDs
- `GET /capabilities/commands` — business command names
- `GET /capabilities/packages/updates` — newest module versions

### Reads (service token only)
| Need | Path (under `/orgs/{org}/`) | Scope |
| --- | --- | --- |
| Who/what am I linked to | `me` | `org:read` |
| Warehouses / locations | `warehouses`, `inventory/…` | `warehouses:read`, `inventory:read` |
| Stock on hand | `inventory` | `inventory:read` |
| Products, variants, categories, brands | `catalog/…` | `catalog:read` |
| Suppliers, POs, receipts | `purchasing/…` | `purchasing:read` |
| Customers | `crm/customers` | `crm:read` |
| Orders | `sales/…` | `sales:read` |
| Chart of accounts, ledger, AR/AP, statements | `accounting/accounts`, `ledger`, `ar`, `ap`, `balance-sheet`, `cash-flow` | `accounting:read` |
| KPIs, reports | `kpis`, `reports/…` | `reports:read` |

### Two write paths (both delegated: the user confirms with the GoOS sign-in popup)
| Kind | Scopes | How | Catalogue |
| --- | --- | --- | --- |
| Master data (products, categories, brands, locations, suppliers, customers, tax rates…) | `catalog:write`, `crm:write`, `finance_master:write`, `locations:write`, `suppliers:write`, `purchasing:write`, `reports:write`, `operations:write` | `POST`/`PATCH` on the master-data route; operation id `<resource>.<create\|update\|archive\|restore>` | `/capabilities/write-operations` (static copy `contracts/master-data-write-operations-1.0.0.json`) |
| Business commands (stock, money, prices, sales) | `inventory:write`, `accounting:write`, `sales:write`, `payments:write`, `pricing:write`, `pos:write` — never on service tokens | `POST /orgs/{org}/commands/{type}` | `/capabilities/commands` |

- **No DELETE.** GoRetailOS answers `405` — records are archived, not deleted. No bulk archive;
  archive each id.
- A new scope needs: the new app's link request list updated, the client's scopes edited in
  GoRetailOS, then a re-approved link.
- GoRetailOS refuses a write unless **all** hold — show a clear message for each: user has a
  linked GoOS identity and an active role in the shop; the link holds the scope; the business
  action is ticked for the link; `ERP_COMMANDS_ENABLED` is on (business commands); grant is fresh
  and matches the exact body.

### Business commands available today
`POST /orgs/{org}/commands/{type}`. Logistics-relevant commands that already exist:

| Real-world event in GoLogisticOS | GoRetailOS command | Effect in GoRetailOS |
| --- | --- | --- |
| Move stock between warehouses | `inventory.transfer.create` → `inventory.transfer.transition` (approve / ship / receive), `inventory.transfer.update_items` | Stock leaves source on ship, arrives at destination on receive |
| Damage / loss / found stock | `inventory.adjustment.post` | Stock movement + atomic balance update |
| Cycle count at a warehouse | `inventory.count.create` / `.save` / `.approve` | Count variances posted |
| Inbound delivery from supplier | `purchasing.po.receive` | Atomic receipt: stock, average cost, PO status, accounting |
| Create / cancel purchase order | `purchasing.po.save`, `purchasing.po.cancel` | PO in Purchasing |
| Delivered a customer order | `sales.order.fulfillment.set` (shipped / delivered / returned) | Order fulfilment status |
| Cash collected on delivery | `sales.order.payment.record` | Payment + journal entry |
| Return picked up | `sales.order.return` | Return, restock, refund accounting |
| Fuel, tolls, driver costs | `accounting.expense.record` (+ `accounting.expense.decide`) | Expense + ledger posting |
| Freight bill from carrier | `purchasing.supplier_invoice.save` / `.post`, `purchasing.supplier_payment.record` | AP + payment |
| Manual journal | `accounting.voucher.create` / `.post` | Ledger |
| New customer / supplier | `crm.customer.create`, `purchasing.supplier.create` | Master data |

High-impact scopes: see `SUMMARY.md` §4 (only via delegated grants).

### How every write is sent
Challenge → fresh GoOS ID token → grant → command with `X-GoOS-Delegation` and `Idempotency-Key`.
Exact steps, body-hash rule and error codes live only in
`contracts/DELEGATED_AUTHORIZATION_CONTRACT.md` (the owner uploads it into the new project; the live `/capabilities` endpoints win if it is stale).

---

## Part 5 — Recipe for each new module in the new app

0. **Pick the module type.**
   | Type | Data lives in | Example |
   | --- | --- | --- |
   | A. Local | The new app only | Driver rota, route plans |
   | B. ERP-backed | GoRetailOS | Warehouse stock screen, expense entry |
   | C. Remote screen | GoRetailOS (screen + data) | A screen GoRetailOS already ships — install the package, don't rebuild |
   | D. Mixed | Own data + GoRetailOS | Shipments that move stock |
   If GoRetailOS already ships the screen, use C.
1. **Decide ownership.** Write two lists: *ERP facts* (owned by GoRetailOS → read/command) and
   *own domain data* (owned by the new app → own tables). Example: Shipment = own;
   its stock movement = GoRetailOS `inventory.transfer.*`.
2. **Check the capability exists:** call `/capabilities/modules` and the commands list. Missing →
   request it in GoRetailOS (Part 7) and stop.
3. **Own tables:** new migration with RLS, policies and explicit least-privilege grants (same
   convention as GoRetailOS). Store GoRetailOS IDs as plain `text/uuid` references + the
   `operation_id` / `idempotency_key` used, never quantities or money balances owned by ERP.
4. **Server functions only** for GoRetailOS calls (`*.server.ts` + `createServerFn`), reusing the
   duplicated connector. Never call GoRetailOS from the browser.
5. **Write flow in the UI:** user clicks → server fn builds body → challenge → fresh ID token →
   grant → command → show "pending" until the operation succeeds → then mark your own row done.
   On failure show the GoRetailOS error message in plain language (raw details only in server
   logs); do not mark done. Never mark stock, postings or payments done locally before GoRetailOS
   confirms. If GoRetailOS is unreachable, keep it pending and retry with the **same** key.
6. **Idempotency:** generate the key once per user action and save it on your row, so retries
   never double-post stock or money.
6a. **React to GoRetailOS events:** status callbacks are signed (verify HMAC) and delivered at
   least once — de-duplicate by event id and apply only newer sequence numbers.
6b. **Package gating:** hide/disable the module with "Install *<package>* first" when the required
   package isn't installed; hide entirely when no shop is connected. Reads may only use endpoints
   the installed package's manifest declares (`apis`); if one is missing, ask GoRetailOS to add it —
   no host-side bypass.
6c. **UI states:** every screen has loading, empty, error and (ERP) pending states. Show names, not
   raw ids; if GoRetailOS doesn't return a name, report it to GoRetailOS. If GoRetailOS is down, return
   a typed fallback (`{ items: [], error }`) and a friendly message.
7. **Show ERP data with installed modules** (Inventory, Accounting …) or fresh reads; do not cache
   balances longer than a page view.
8. **Tests:** with a mocked ERP client cover validation, permissions, other-tenant denial,
   module-off hiding, package/scope gating, idempotency-key reuse on retry, pending state and the
   refused DELETE; unit-test body hashing and mapping; end-to-end only on a synthetic GoRetailOS test shop,
   never a live shop. Verify the effect is visible in GoRetailOS portal and in the installed module.
9. **Docs:** add the module to the new app's roadmap and to the ownership table.

### Example — "Dispatch shipment" in GoLogisticOS
1. User creates shipment (own table): source warehouse, destination, lines (variant IDs, qty).
2. Click **Dispatch** → `inventory.transfer.create` then `inventory.transfer.transition` → `shipped`.
3. Save returned transfer ID + idempotency key on the shipment.
4. On **Delivered** → `inventory.transfer.transition` → `received`; for a customer order also
   `sales.order.fulfillment.set` → `delivered`, and `sales.order.payment.record` if cash collected.
5. Fuel cost → `accounting.expense.record`.
Result: GoRetailOS Inventory, Transfers, Orders and Accounting (and the same modules installed in
GoCoreOS/GoLogisticOS) show the change immediately.

---

## Part 6 — First prompt to paste into the new project

```text
This project (GoLogisticOS) started as a private duplicate of GoCoreOS and uses GoRetailOS as its ERP.
You cannot read the GoRetailOS or GoCoreOS repos; only the files uploaded here and the live HTTPS APIs.
Read docs/GORETAILOS_ERP_RULES.md first and follow it strictly.

Rules:
- GoRetailOS owns all inventory, warehouse, catalog, purchasing, sales, customer and accounting
  records. Never create tables for those or store their quantities/balances here.
- Every ERP change goes through a GoRetailOS command via the existing connector, using the
  delegated flow (challenge -> fresh GoOS ID token -> grant -> command) with an Idempotency-Key.
- All GoRetailOS calls are server-side. Discover modules/commands from /capabilities; never
  hard-code versions or routes.
- Keep the duplicated SSO, connector, Marketplace and module loader; replace GoCoreOS-specific
  secrets with new GoLogisticOS ones (new GoOS client, new ERP client). Never reuse GoCoreOS secrets.
- If a needed GoRetailOS command does not exist, stop and report it instead of building one.
- Never test against a live shop; use a synthetic GoRetailOS test shop.

Step 1 (report only, no code): list what came over from GoCoreOS (SSO, connector, Marketplace, loader,
callback), what must be renamed/re-keyed for GoLogisticOS, and an ownership table for the first
module "Shipments" (own data vs GoRetailOS commands from section 4).
```

---

## Part 7 — When GoRetailOS needs something new

The new app cannot see GoRetailOS code. Write the request as text; the owner pastes it into the GoRetailOS project: *"Add ERP command `<name>` for `<event>`, reusing service
`<existing service>`, scope `<scope>`."* GoRetailOS then (per its rules): adds a domain command that
calls the existing service, adds it to a package, releases a new co-versioned module version,
publishes; the platform admin ticks the new business action on each link. Stock reservations and
order creation from external apps are **not available yet** — plan around them.

## Part 8 — Checklist before go-live
- [ ] Own private repo + own Lovable project; rules/contracts uploaded and matching live `/capabilities`
- [ ] Own GoOS client + own ERP client + own secrets
- [ ] Link active with only the needed scopes; business actions ticked
- [ ] Users linked via "Link my company account"
- [ ] Catalog installed first; all packages same version
- [ ] Every write uses challenge/grant + Idempotency-Key; no ERP tables in the new app
- [ ] Tested on a synthetic shop; effect visible in GoRetailOS and installed modules
- [ ] GoRetailOS `ERP_API_ENABLED` / `ERP_COMMANDS_ENABLED` switched on only with owner approval

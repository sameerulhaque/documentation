# GoRetailOS → GoCoreOS — Integration Guide (how it works, how we set it up, how to add more)

Plain-language, step-by-step guide. Technical reference: `SUMMARY.md` in this folder and the
contracts in `contracts/`.

---

## Part 1 — How it works (the picture)

```text
 User ──signs in──> GoOS Identity (central sign-in app, go-oss-passport.lovable.app)
   │                       │  confirms "this is Sameer"
   ▼                       ▼
 GoCoreOS (host)  ──server-to-server──>  GoRetailOS (provider, owns all data and rules)
   - menu/shell                          - ERP API  /api/erp/v1/...
   - proxy                               - module screens  /erp-remotes/<module>/<version>/
   - module loader  <──screen files──    - checks every write again
```

- **GoRetailOS is the source of truth.** Products, stock, orders, accounts live only here. Every
  rule (stock, tax, prices, permissions) runs here.
- **GoCoreOS shows GoRetailOS screens inside its own menu.** The screens are the *same* screens as
  the GoRetailOS admin portal, delivered as module files. No iframe, no copied logic.
- **GoOS Identity** lets one person use the same login in every Go*OS app. Each app still keeps its
  own users and roles — a central login never grants permissions by itself.
- **Three safety switches** (server settings in GoRetailOS, off by default):
  - `SSO_ENABLED` — "Sign in with GoOS" / account linking
  - `ERP_API_ENABLED` — the whole ERP connection (off = everything returns "not found")
  - `ERP_COMMANDS_ENABLED` — business actions that change data
- **Live shops:** never test on a real paying shop. `LICENSING_ENFORCED` stays off in production.

### What a "read" does
GoCoreOS server asks GoRetailOS with its service token → GoRetailOS checks the token, the link and
its scopes → returns data for the linked shop only (shop is taken from the link, never from the
request).

### What a "write" does (e.g. edit a product)
GoCoreOS proves *which person* clicked, GoRetailOS re-checks that person's shop role, then runs the
same service the portal uses. A stolen token cannot be reused. Technical steps: `SUMMARY.md` §5 and
`contracts/DELEGATED_AUTHORIZATION_CONTRACT.md`.

---

## Part 2 — What we did, step by step (current setup)

### Stage A — Central sign-in (done once, by the owner)
1. **Central identity project (Supabase Pro):** turned on the OAuth server and asymmetric signing
   keys.
2. **GoOS Identity app** built (`go-oss-passport.lovable.app`) — shows the sign-in and the
   "Allow GoRetailOS to …" consent page.
3. **Registered GoRetailOS** as a confidential app on the identity project with callback
   `https://goretailos.com/api/sso/callback`. Its ID/secret were saved as GoRetailOS secrets
   (`SSO_CLIENT_ID`, `SSO_CLIENT_SECRET`, `SSO_ISSUER_URL`).
4. **Registered GoCoreOS** as a separate app on the identity project → it got its **own** client
   ID. Keep this ID — it is the "Sign-in app ID" used in Stage C.
5. Turned `SSO_ENABLED` on and published GoRetailOS.

### Stage B — Link your own account (each user, in GoRetailOS)
1. GoRetailOS → **Admin → ERP Connections** (`/admin/erp-links`).
2. Panel **Company single sign-on** → click **Link my company account**.
3. Sign in on GoOS Identity → click **Allow** on the consent page.
4. Back in GoRetailOS the panel lists the linked identity (e.g. your email) with **Unlink**.
   Writes from GoCoreOS only work for users who did this.

### Stage C — Register GoCoreOS as an ERP client (platform admin, in GoRetailOS)
On **Admin → ERP Connections**, panel **ERP clients (platform)**:
1. Type **Client name** = `GoCoreOS`, tick scopes, click **Register client**.
2. Copy the **client_secret** shown once (it is never shown again; use **Rotate** to get a new one).
   Give client ID + secret to GoCoreOS as secrets.
3. On the client card:
   - **Edit scopes → Select all scopes → Save** (GoCoreOS needs all 22).
   - **Status callback URL** = GoCoreOS's callback address → **Save callback**.
   - **Host app's GoOS Identity client ID** = GoCoreOS's identity client ID from Stage A step 4
     (not the ERP client ID) → **Save sign-in app ID**.

### Stage D — GoCoreOS settings (in the GoCoreOS project)
Add secrets in GoCoreOS:
| Secret | Value |
| --- | --- |
| `GORETAILOS_ERP_ENABLED` | `true` |
| `GORETAILOS_ERP_BASE_URL` | `https://goretailos.com` |
| `GORETAILOS_ERP_CLIENT_ID` / `_SECRET` | from Stage C step 2 |
| `GORETAILOS_CALLBACK_SECRET` | shared secret for signed status callbacks |
Also its own GoOS sign-in client ID/secret from Stage A step 4.

### Stage E — Connect an organisation to a shop
1. **GoCoreOS → Admin → ERP Connections → Connect GoRetailOS.** GoCoreOS requests all scopes and
   shows an **approval code** (`XXXX-XXXX`).
2. **GoRetailOS → Admin → ERP Connections** (as shop owner/admin, with the right shop selected):
   type the code in **Approval code** → **Approve**. Message: "Approved GoCoreOS … GoCoreOS must
   now confirm."
3. **GoCoreOS** confirms → the link shows **active** in both apps with the full scope list.
   - If GoCoreOS says *"Requested scopes are not allowed"*: the client card is missing scopes →
     redo Stage C step 3, then connect again.
   - A link keeps the scopes it was approved with. To add scopes, **Revoke** and connect again.

### Stage F — Allow business actions on the link (platform admin, in GoRetailOS)
Panel **Business actions per link (platform)** → for the link click **Select all** (or tick single
actions) → saves in one go. Today: 64 actions. Actions still need `ERP_COMMANDS_ENABLED` on.

### Stage G — Switch on and publish
1. Publish GoRetailOS (so the newest module files are live).
2. Set `ERP_API_ENABLED=true`; when ready for writes, `ERP_COMMANDS_ENABLED=true`.
3. Check panel **Integration operations** for failed operations/callbacks → **Retry** if needed.

### Stage H — Install modules in GoCoreOS
1. **GoCoreOS → Admin → ERP Modules → Marketplace.**
2. Install **Catalog & Products** (`goretailos.catalog`) **first** — others depend on it.
3. Install the rest: Inventory & Purchasing, CRM & Loyalty, Sales & Orders, Accounting & Finance,
   Dashboards & Reports, KPI Builder.
4. Keep all seven on the **same version** (today **1.16.0**); upgrade/downgrade all together.
5. Open a screen (e.g. Categories) → it should look like the GoRetailOS portal.

### Stage I — Updates after that
- From 1.16.0, each module checks for a newer safe version, verifies it and swaps itself in; if it
  fails it goes back to the old version. Big/breaking versions wait for a manual upgrade in GoCoreOS.
- Emergency rollback for everyone: see `SUMMARY.md` §7 (release script).

### Problems we hit and fixes (keep for reference)
| Symptom | Cause | Fix |
| --- | --- | --- |
| Install button disabled | No active link / Catalog not installed | Finish Stage E, install Catalog first |
| "Module unavailable (404)" | GoCoreOS proxy only allowed old versions | GoCoreOS reads versions from the live catalogue |
| "Module needs unsupported imports" | GoCoreOS checker read text inside code as imports | Fixed in GoCoreOS checker |
| Raw IDs / dashes in tables | Old generic screens | Shared portal screens (1.5.0+) |
| Business actions missing | Link revoked | Reconnect and approve |
| Scope list short after approval | GoCoreOS requested only a few scopes | GoCoreOS must request all scopes |

### Turning it off
Set `ERP_API_ENABLED=false` (instant) → **Revoke** links → **Rotate** client secrets. No business
data is deleted. Uninstalling a module in GoCoreOS never deletes data.

---

## Part 3 — Integrating more in future (recipe)

### Add a new GoRetailOS screen/module to GoCoreOS
1. The screen must already work in the GoRetailOS portal and read/write through server services
   (`*.server.ts`), not directly from the browser.
2. Register the module in `src/lib/erp/erp-modules.ts` (registry served at `/capabilities/modules`).
3. Map it to its portal route in the shared-screen list (`src/components/screens/shared-screens.ts`)
   and add any needed adapter under `src/lib/screens/remote-shims/`.
4. Reads: add a read endpoint in the ERP API with the right `*:read` scope.
5. Writes: add a domain command in `erp-commands.ts` that calls the **existing** service (never a
   second engine). Its form appears automatically from its input schema.
6. Add it to the right package in `erp-package.ts`.
7. Release: `bun scripts/erp/release.ts` → `--check` must be clean → tests pass.
8. Publish GoRetailOS → modules update themselves in GoCoreOS (or upgrade all in Marketplace).
9. Platform admin ticks the new business action on each link (Stage F).

### Add a new database table for it
Same migration must include RLS, policies and explicit least-privilege grants
(`docs/supabase-migration/DATA_API_GRANT_CONVENTION.md`). ERP tables are server-only.

### Connect another Go*OS app (not GoCoreOS)
See `NEW_GOOS_APP_ON_GORETAILOS_ERP.md`.

### Still to do
See `SUMMARY.md` §9 and root `roadmap.md`.

# GoRetailOS — Master QA Test Specification

**Application:** GoRetailOS / Retail OS — AI-Powered Multi-Tenant Retail Operating System
**Purpose:** Authoritative, code-grounded, execution-oriented test specification for a future Playwright automation chat.
**Status:** Phase 1 complete. This spec is derived from the current source code (`src/`), the Supabase schema/migrations (`supabase/`), and live verification against `https://goretailos.com`.

> **How to use this file:** A future automation chat should read this entire document, then implement Playwright tests. Every test case has a stable ID, concrete selectors (actual rendered labels/headings found in code), explicit preconditions, expected backend/data results, negative assertions, and cleanup. Do not renumber IDs. Use the `QA-` prefix for all test-created records so automation can find and clean its own data.

---

## 1. Test Data Strategy

### 1.1 QA Shops

| Shop | Slug | Purpose |
| --- | --- | --- |
| **QA Shop A** | `qa-shop-a` (or existing `trendies-store`) | Primary shop for normal functional tests. |
| **QA Shop B** | `qa-shop-b` | Second tenant for tenant-isolation tests. Must be a *separate* shop with its own owner. |

> **Live environment note:** The production tenant `trendies-store` (owner `sarimulhaq74@gmail.com`) is used for live verification. For automated runs, prefer creating dedicated `QA-` shops via onboarding, or use the existing tenant with `QA-`-prefixed records that are cleaned up after the run. **Never reset or delete production/shared data.**

### 1.2 Users / Personas

Credentials are **never** committed. Define them as environment variables:

| Persona | Env var (email) | Env var (password) | Role |
| --- | --- | --- | --- |
| Owner | `E2E_OWNER_EMAIL` | `E2E_OWNER_PASSWORD` | owner |
| Admin | `E2E_ADMIN_EMAIL` | `E2E_ADMIN_PASSWORD` | admin |
| Manager | `E2E_MANAGER_EMAIL` | `E2E_MANAGER_PASSWORD` | manager |
| Cashier | `E2E_CASHIER_EMAIL` | `E2E_CASHIER_PASSWORD` | cashier |
| Accountant | `E2E_ACCOUNTANT_EMAIL` | `E2E_ACCOUNTANT_PASSWORD` | accountant |
| Inventory | `E2E_INVENTORY_EMAIL` | `E2E_INVENTORY_PASSWORD` | inventory |
| Staff | `E2E_STAFF_EMAIL` | `E2E_STAFF_PASSWORD` | staff |
| Customer | `E2E_CUSTOMER_EMAIL` | `E2E_CUSTOMER_PASSWORD` | customer (storefront) |
| Guest | — | — | anonymous storefront visitor |

**Live owner account (for manual/live verification only):** `sarimulhaq74@gmail.com` / `19972015AH` (owner of `trendies-store`). Do not commit this to the repo.

### 1.3 Reusable Test Entities

Use a consistent `QA-` prefix, ideally with a run ID: `QA-${RUN_ID}-Product`. Examples:

- Product: `QA Perfume 100ml`, SKU `QA-PERF-100`, price `100`, cost `60`, tax `0%`, opening stock `10`
- Customer: `QA Customer One`, phone `+92 300 0000001`, email `qa.customer1@example.com`
- Supplier: `QA Supplier One`, email `qa.supplier1@example.com`
- Category: `QA Category`
- Brand: `QA Brand`
- Register: `QA Register 1`
- Branch: `QA Branch 1`
- Location: `QA Location 1`

### 1.4 Environment Variables for Playwright

- `PLAYWRIGHT_BASE_URL` (default `http://localhost:8080`; live `https://goretailos.com`)
- `PW_SHOP_SLUG` (default `trendies-store`)
- `PW_SHOP_ID` (the shop UUID for the active shop)
- `PW_EMAIL` / `PW_PASSWORD` (existing staff auth setup)
- `E2E_*` persona variables (see 1.2)

---

## 2. Authentication Test Cases

### AUTH-001 — Staff sign in (happy path)
- **Area:** Authentication
- **Title:** Owner signs in with valid email/password
- **Priority:** P0
- **Test Type:** Smoke, Happy Path
- **Suitable for Playwright?** Yes
- **Roles:** Owner
- **Preconditions:** Owner account exists; not already authenticated.
- **Test Data:** `E2E_OWNER_EMAIL` / `E2E_OWNER_PASSWORD`
- **Route:** `/auth`
- **Selectors / UI Targets:**
  - textbox `Email` (placeholder `Email`)
  - textbox `Password` (placeholder `Password`)
  - button `Sign in`
  - heading `Staff sign in`
- **Steps:**
  1. Navigate to `/auth`.
  2. Assert heading `Staff sign in` is visible.
  3. Fill `Email` and `Password`.
  4. Click `Sign in`.
- **Expected UI Result:** Redirects to `/admin` (or `/onboarding` if no shop). Admin shell renders.
- **Expected Backend / Data Result:** A Supabase session is created; `shop_users` membership is loaded.
- **Side Effects:** None beyond session.
- **Negative Assertions:** Must not stay on `/auth` with no error.
- **Cleanup:** Sign out.
- **Notes for Playwright:** `/auth` is server-rendered; the form is clickable before React attaches handlers. Retry the submit until navigation or an error appears (see existing `auth.setup.ts`). Use `page.getByRole('button', { name: 'Sign in', exact: true })`.

### AUTH-002 — Sign in with invalid password
- **Area:** Authentication
- **Title:** Invalid password is rejected with a clear message
- **Priority:** P1
- **Test Type:** Negative, Validation
- **Suitable for Playwright?** Yes
- **Roles:** Owner
- **Preconditions:** Owner account exists.
- **Test Data:** `E2E_OWNER_EMAIL` + wrong password `wrong-password-123`
- **Route:** `/auth`
- **Selectors / UI Targets:** textbox `Email`, textbox `Password`, button `Sign in`, error paragraph `p.text-destructive`
- **Steps:**
  1. Navigate to `/auth`.
  2. Fill email with `E2E_OWNER_EMAIL`, password with `wrong-password-123`.
  3. Click `Sign in`.
- **Expected UI Result:** An error message appears: `That email and password combination doesn't match an account.`
- **Expected Backend / Data Result:** No session created.
- **Negative Assertions:** Must not navigate to `/admin`.
- **Cleanup:** None.
- **Notes for Playwright:** Assert the error text via `page.locator('p.text-destructive')`.

### AUTH-003 — Sign in with unknown account
- **Area:** Authentication
- **Title:** Unknown email is rejected
- **Priority:** P2
- **Test Type:** Negative
- **Suitable for Playwright?** Yes
- **Roles:** Guest
- **Preconditions:** None.
- **Test Data:** `no-such-user-${RUN_ID}@example.com` / `whatever123`
- **Route:** `/auth`
- **Selectors / UI Targets:** textbox `Email`, textbox `Password`, button `Sign in`, error paragraph
- **Steps:**
  1. Navigate to `/auth`.
  2. Fill unknown email + password.
  3. Click `Sign in`.
- **Expected UI Result:** Error message appears (invalid login).
- **Expected Backend / Data Result:** No session.
- **Negative Assertions:** No navigation to `/admin`.
- **Cleanup:** None.

### AUTH-004 — Sign in with empty fields
- **Area:** Authentication
- **Title:** Empty email/password does not submit
- **Priority:** P2
- **Test Type:** Validation
- **Suitable for Playwright?** Yes
- **Roles:** Guest
- **Preconditions:** None.
- **Route:** `/auth`
- **Selectors / UI Targets:** button `Sign in`
- **Steps:**
  1. Navigate to `/auth`.
  2. Leave fields empty.
  3. Click `Sign in`.
- **Expected UI Result:** Supabase returns an error (e.g. "Email is required") shown in the error paragraph, OR the button is disabled. No navigation.
- **Expected Backend / Data Result:** No session.
- **Negative Assertions:** No navigation to `/admin`.
- **Cleanup:** None.

### AUTH-005 — Sign out
- **Area:** Authentication
- **Title:** Owner signs out and is redirected to `/auth`
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?** Yes
- **Roles:** Owner
- **Preconditions:** Owner authenticated.
- **Route:** `/admin`
- **Selectors / UI Targets:** button `Account` (profile menu), then sign-out option.
- **Steps:**
  1. Authenticate as owner.
  2. Open the account/profile menu (button `Account`).
  3. Click sign out.
- **Expected UI Result:** Redirected to `/auth` (or landing). Session cleared.
- **Expected Backend / Data Result:** Supabase session invalidated.
- **Negative Assertions:** Navigating to `/admin` after sign-out redirects to `/auth`.
- **Cleanup:** None.

### AUTH-006 — Unauthenticated access to protected route redirects to `/auth`
- **Area:** Authentication
- **Title:** Guest visiting `/admin` is redirected to `/auth`
- **Priority:** P0
- **Test Type:** Security, Negative
- **Suitable for Playwright?** Yes
- **Roles:** Guest
- **Preconditions:** No session.
- **Route:** `/admin`
- **Selectors / UI Targets:** none (redirect)
- **Steps:**
  1. As a guest (no storage state), navigate to `/admin`.
- **Expected UI Result:** Redirected to `/auth`.
- **Expected Backend / Data Result:** `_authenticated` route `beforeLoad` throws redirect.
- **Negative Assertions:** Must not render admin content.
- **Cleanup:** None.

### AUTH-007 — Authenticated user visiting `/auth` is redirected to `/admin`
- **Area:** Authentication
- **Title:** Signed-in user visiting `/auth` goes to `/admin`
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?** Yes
- **Roles:** Owner
- **Preconditions:** Owner authenticated.
- **Route:** `/auth`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/auth`.
- **Expected UI Result:** Redirected to `/admin` (the `useEffect` in `auth.tsx` navigates if a user exists).
- **Expected Backend / Data Result:** None.
- **Negative Assertions:** Must not show the sign-in form.
- **Cleanup:** None.

### AUTH-008 — Sign up creates an account
- **Area:** Authentication
- **Title:** New staff account sign-up
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?** Yes (requires email confirmation handling)
- **Roles:** Guest
- **Preconditions:** Email not already registered.
- **Test Data:** `qa.newstaff.${RUN_ID}@example.com` / `Password123!`
- **Route:** `/auth?mode=signup`
- **Selectors / UI Targets:** textbox `Full name`, textbox `Email`, textbox `Password`, button `Create account`
- **Steps:**
  1. Navigate to `/auth?mode=signup`.
  2. Fill `Full name`, `Email`, `Password`.
  3. Click `Create account`.
- **Expected UI Result:** Either navigates to `/admin` (if session auto-created) or shows `Check your email to confirm your account, then sign in.`
- **Expected Backend / Data Result:** A `profiles` row is created; a Supabase auth user exists.
- **Negative Assertions:** If email already registered, shows `An account with that email already exists — sign in instead.`
- **Cleanup:** Delete the created user via admin API if possible.
- **Notes for Playwright:** Email confirmation may be required; handle both outcomes.

### AUTH-009 — Google sign-in button present
- **Area:** Authentication
- **Title:** Google OAuth button renders
- **Priority:** P3
- **Test Type:** Smoke
- **Suitable for Playwright?:** Partial (OAuth flow is external)
- **Roles:** Guest
- **Route:** `/auth`
- **Selectors / UI Targets:** button `Continue with Google`
- **Steps:**
  1. Navigate to `/auth`.
  2. Assert `Continue with Google` button is visible.
- **Expected UI Result:** Button visible.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.
- **Notes for Playwright:** Do not complete the OAuth flow in automated tests (external provider).

### AUTH-010 — Forgot password link present and flow
- **Area:** Authentication
- **Title:** Forgot password request
- **Priority:** P3
- **Test Type:** Smoke
- **Suitable for Playwright?:** Partial (email delivery external)
- **Roles:** Guest
- **Route:** `/auth`
- **Selectors / UI Targets:** button `Forgot password?`
- **Steps:**
  1. Navigate to `/auth`.
  2. Click `Forgot password?`.
- **Expected UI Result:** A password reset email is requested (message shown).
- **Expected Backend / Data Result:** `resetPasswordForEmail` called.
- **Cleanup:** None.

---

## 3. Onboarding Test Cases

### ONBOARD-001 — First-time user onboarding creates a shop
- **Area:** Onboarding
- **Title:** New user provisions a shop with all dependent records
- **Priority:** P0
- **Test Type:** Happy Path, Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (new)
- **Preconditions:** A fresh user with no shops; authenticated.
- **Test Data:** Shop name `QA Shop ${RUN_ID}`, slug `qa-shop-${RUN_ID}`, currency `PKR`, timezone `Asia/Karachi`, locale `en`.
- **Route:** `/onboarding`
- **Selectors / UI Targets:**
  - field `Shop name`
  - field `Store address (URL slug)`
  - select `Currency`
  - select `Timezone`
  - select `Default language`
  - button `Create shop`
- **Steps:**
  1. Authenticate as a user with no shops.
  2. Navigate to `/onboarding`.
  3. Fill `Shop name`, `Store address`, select `Currency`, `Timezone`, `Default language`.
  4. Click `Create shop`.
- **Expected UI Result:** Redirects to `/admin`.
- **Expected Backend / Data Result:** `provisionShop` creates:
  - `shops` row (with `DEFAULT_SHOP_SETTINGS`)
  - `shop_users` row (role `owner`)
  - `branches` row (`Main Branch`, `MAIN`, default)
  - `registers` row (`Register 1`, `REG1`)
  - `inventory_locations` row (`Main Store`, default)
  - `payment_methods` (Cash, Card, Bank Transfer, Store Credit disabled, COD)
  - `tax_rates` row (`Standard VAT`, 0%, default)
  - `accounts` (full chart of accounts: 1000, 1100, 1150, 1200, 2000, 2100, 3000, 4000, 4100, 5000, 6000, 6100, 6200, 6300)
  - `document_sequences` (order, receipt, invoice, purchase_order, return, credit_note, debit_note, goods_receipt, product)
  - `ai_permissions` (all salesman + assistant capabilities)
  - `storefront_settings`, `storefront_navigation`, `storefront_policies`, `expense_categories`
- **Side Effects:** Owner membership created.
- **Negative Assertions:** No duplicate shop; no missing dependent records.
- **Cleanup:** Delete the QA shop and its dependent rows (cascade).
- **Notes for Playwright:** Verify dependent records via DB assertions after onboarding.

### ONBOARD-002 — Duplicate shop slug rejected
- **Area:** Onboarding
- **Title:** Creating a shop with an existing slug fails
- **Priority:** P1
- **Test Type:** Negative, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A shop with slug `trendies-store` exists.
- **Test Data:** Shop name `QA Dupe`, slug `trendies-store`.
- **Route:** `/onboarding`
- **Steps:**
  1. Authenticate as a user with no shops.
  2. Navigate to `/onboarding`.
  3. Fill name and set slug to `trendies-store`.
  4. Click `Create shop`.
- **Expected UI Result:** Error `That store address is already taken`.
- **Expected Backend / Data Result:** No shop created.
- **Negative Assertions:** No duplicate slug.
- **Cleanup:** None.

### ONBOARD-003 — Onboarding requires a shop name
- **Area:** Onboarding
- **Title:** Create shop button disabled without a name
- **Priority:** P2
- **Test Type:** Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/onboarding`
- **Selectors / UI Targets:** button `Create shop`
- **Steps:**
  1. Navigate to `/onboarding`.
  2. Leave `Shop name` empty.
- **Expected UI Result:** `Create shop` button is disabled (`disabled={busy || form.name.length < 2}`).
- **Expected Backend / Data Result:** No request.
- **Cleanup:** None.

### ONBOARD-004 — Onboarding double-submit protection
- **Area:** Onboarding
- **Title:** Double-clicking Create shop does not create two shops
- **Priority:** P1
- **Test Type:** Concurrency
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/onboarding`
- **Steps:**
  1. Navigate to `/onboarding`.
  2. Fill valid data.
  3. Click `Create shop` twice rapidly.
- **Expected UI Result:** Only one shop created; button shows `Creating…` and is disabled while busy.
- **Expected Backend / Data Result:** Exactly one `shops` row.
- **Negative Assertions:** No duplicate shop.
- **Cleanup:** Delete the QA shop.

---

## 4. Multi-Tenancy / Tenant Isolation Test Cases (P0)

> **Critical:** These tests must run as the correct tenant persona. Use `shopAOwner` / `shopBOwner` fixtures. Use deterministic `QA-SHOPA-ONLY-*` and `QA-SHOPB-ONLY-*` records.

### TENANT-001 — Shop A cannot see Shop B products in the admin catalog
- **Area:** Multi-tenancy
- **Title:** Products are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** Shop A and Shop B exist. A product `QA-SHOPB-ONLY-PRODUCT` exists only in Shop B.
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/products`.
  3. Search for `QA-SHOPB-ONLY-PRODUCT`.
- **Expected UI Result:** The Shop B product is not listed.
- **Expected Backend / Data Result:** The products query filters by `shop_id = Shop A`.
- **Negative Assertions:** Shop B product must not appear.
- **Cleanup:** None (read-only).
- **Notes for Playwright:** Seed `QA-SHOPB-ONLY-PRODUCT` in Shop B via DB setup.

### TENANT-002 — Shop A cannot access Shop B product by direct URL/ID
- **Area:** Multi-tenancy
- **Title:** Direct product detail access is tenant-scoped
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B product with a known ID exists.
- **Route:** `/admin/products` (via product detail modal)
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Attempt to load Shop B product detail by its ID (via API or URL manipulation).
- **Expected UI Result:** Product not found / not accessible.
- **Expected Backend / Data Result:** `getProductDetail` filters by `shop_id = Shop A`; returns null for Shop B product.
- **Negative Assertions:** Shop B product data must not leak.
- **Cleanup:** None.

### TENANT-003 — Shop A cannot see Shop B orders
- **Area:** Multi-tenancy
- **Title:** Orders are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B order exists.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/orders`.
  3. Search for the Shop B order number.
- **Expected UI Result:** The Shop B order is not listed.
- **Expected Backend / Data Result:** Orders query filters by `shop_id = Shop A`.
- **Negative Assertions:** Shop B order must not appear.
- **Cleanup:** None.

### TENANT-004 — Shop A cannot access Shop B order by direct ID
- **Area:** Multi-tenancy
- **Title:** Direct order detail access is tenant-scoped
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B order with a known ID exists.
- **Route:** `/admin/sales` (sale detail)
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Attempt to load Shop B order detail by ID.
- **Expected UI Result:** `Sale not found`.
- **Expected Backend / Data Result:** `getSaleDetail` filters by `shop_id = Shop A`.
- **Negative Assertions:** Shop B order data must not leak.
- **Cleanup:** None.

### TENANT-005 — Shop A cannot see Shop B customers
- **Area:** Multi-tenancy
- **Title:** Customers are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B customer exists.
- **Route:** `/admin/customers`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/customers`.
  3. Search for the Shop B customer.
- **Expected UI Result:** The Shop B customer is not listed.
- **Expected Backend / Data Result:** `listCustomers` filters by `shop_id = Shop A`.
- **Negative Assertions:** Shop B customer must not appear.
- **Cleanup:** None.

### TENANT-006 — Shop A cannot see Shop B suppliers
- **Area:** Multi-tenancy
- **Title:** Suppliers are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B supplier exists.
- **Route:** `/admin/suppliers`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/suppliers`.
  3. Search for the Shop B supplier.
- **Expected UI Result:** The Shop B supplier is not listed.
- **Expected Backend / Data Result:** Suppliers query filters by `shop_id = Shop A`.
- **Negative Assertions:** Shop B supplier must not appear.
- **Cleanup:** None.

### TENANT-007 — Shop A cannot see Shop B inventory
- **Area:** Multi-tenancy
- **Title:** Inventory balances are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B inventory balance exists.
- **Route:** `/admin/inventory`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/inventory`.
- **Expected UI Result:** Only Shop A stock is shown.
- **Expected Backend / Data Result:** Inventory queries filter by `shop_id = Shop A`.
- **Negative Assertions:** Shop B stock must not appear.
- **Cleanup:** None.

### TENANT-008 — Shop A cannot see Shop B accounting
- **Area:** Multi-tenancy
- **Title:** Journal entries are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B journal entry exists.
- **Route:** `/admin/accounting`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/accounting`.
  3. Open the General Ledger or Trial Balance.
- **Expected UI Result:** Only Shop A ledger data is shown.
- **Expected Backend / Data Result:** Accounting queries filter by `shop_id = Shop A`.
- **Negative Assertions:** Shop B ledger data must not appear.
- **Cleanup:** None.

### TENANT-009 — Shop A cannot see Shop B audit logs
- **Area:** Multi-tenancy
- **Title:** Audit logs are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B audit log exists.
- **Route:** `/admin/audit`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/audit`.
- **Expected UI Result:** Only Shop A audit entries are shown.
- **Expected Backend / Data Result:** `listAuditLogs` filters by `shop_id = Shop A`.
- **Negative Assertions:** Shop B audit entries must not appear.
- **Cleanup:** None.

### TENANT-010 — Shop A cannot see Shop B payment proofs
- **Area:** Multi-tenancy
- **Title:** Payment proofs are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B payment proof exists.
- **Route:** `/admin/orders` (payment proof section)
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Attempt to fetch a Shop B payment proof by ID.
- **Expected UI Result:** `Proof not found`.
- **Expected Backend / Data Result:** `getPaymentProofUrl` filters by `shop_id = Shop A`.
- **Negative Assertions:** Shop B proof must not be accessible.
- **Cleanup:** None.

### TENANT-011 — Shop A cannot see Shop B storefront admin content
- **Area:** Multi-tenancy
- **Title:** Storefront CMS content is isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B banner/section/page exists.
- **Route:** `/admin/storefront`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/storefront`.
- **Expected UI Result:** Only Shop A storefront content is shown.
- **Expected Backend / Data Result:** Storefront admin queries filter by `shop_id = Shop A`.
- **Negative Assertions:** Shop B content must not appear.
- **Cleanup:** None.

### TENANT-012 — Shop A cannot see Shop B AI data
- **Area:** Multi-tenancy
- **Title:** AI conversations/knowledge are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B AI conversation/knowledge doc exists.
- **Route:** `/admin/ai`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/ai`.
  3. Open the Knowledge or Memory tab.
- **Expected UI Result:** Only Shop A AI data is shown.
- **Expected Backend / Data Result:** AI queries filter by `shop_id = Shop A`.
- **Negative Assertions:** Shop B AI data must not appear.
- **Cleanup:** None.

### TENANT-013 — Shop A cannot see Shop B custom domains
- **Area:** Multi-tenancy
- **Title:** Custom domains are isolated per shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B domain exists.
- **Route:** `/admin/storefront` (Domains tab)
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/storefront` → Domains tab.
- **Expected UI Result:** Only Shop A domains are shown.
- **Expected Backend / Data Result:** `listDomains` filters by `shop_id = Shop A`.
- **Negative Assertions:** Shop B domains must not appear.
- **Cleanup:** None.

### TENANT-014 — Shop A cannot see Shop B data manager tables
- **Area:** Multi-tenancy
- **Title:** Data manager is tenant-scoped
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** Shop A and Shop B exist.
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/data-manager`.
  3. Preview a table.
- **Expected UI Result:** Only Shop A rows are shown.
- **Expected Backend / Data Result:** `dmPreview` forces `.eq("shop_id", Shop A)`.
- **Negative Assertions:** Shop B rows must not appear.
- **Cleanup:** None.

### TENANT-015 — Shop A cannot insert a Shop B shop_id via data manager
- **Area:** Multi-tenancy
- **Title:** Data manager insert always stamps the caller's shop_id
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** Shop A and Shop B exist.
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/data-manager`.
  3. Attempt to insert a row with `shop_id` = Shop B's ID.
- **Expected UI Result:** The inserted row is stamped with Shop A's `shop_id` (the client-supplied `shop_id` is dropped).
- **Expected Backend / Data Result:** `dmInsert` strips `shop_id` from the payload and stamps `shopId`.
- **Negative Assertions:** No row is created in Shop B.
- **Cleanup:** Delete the inserted QA row.

### TENANT-016 — Shop switching reloads data and clears old shop state
- **Area:** Multi-tenancy
- **Title:** Switching shops updates the active shop and reloads data
- **Priority:** P0
- **Test Type:** Tenant Isolation, Integration
- **Suitable for Playwright?:** Yes
- **Roles:** Owner (multi-shop user)
- **Preconditions:** A user belongs to both Shop A and Shop B.
- **Route:** `/admin`
- **Selectors / UI Targets:** shop switcher button (shows current shop name), e.g. `T Trendies Store (Haroon Shopping Center)`.
- **Steps:**
  1. Authenticate as a multi-shop owner.
  2. Note the current shop in the header.
  3. Open the shop switcher and select the other shop.
- **Expected UI Result:** Header shows the new shop; dashboard KPIs reload for the new shop.
- **Expected Backend / Data Result:** `setShopId` writes `retail-os:active-shop` to localStorage; queries refetch with the new `shop_id`.
- **Negative Assertions:** Old shop data must not remain visible after switching.
- **Cleanup:** Switch back to the original shop.

### TENANT-017 — Storefront of Shop A does not expose Shop B products
- **Area:** Multi-tenancy
- **Title:** Public storefront is tenant-scoped
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** Shop A and Shop B exist with distinct products.
- **Route:** `/store/qa-shop-a/shop`
- **Steps:**
  1. As a guest, navigate to Shop A's storefront shop page.
  2. Search for a Shop B-only product.
- **Expected UI Result:** The Shop B product is not shown.
- **Expected Backend / Data Result:** `listStorefrontProducts` filters by `shop_id = Shop A`.
- **Negative Assertions:** Shop B product must not appear.
- **Cleanup:** None.

### TENANT-018 — Storefront of Shop A does not expose Shop B cost prices
- **Area:** Multi-tenancy
- **Title:** Cost price never leaks across tenants or to the public
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** A product with a cost price exists.
- **Route:** `/store/qa-shop-a/product/<slug>`
- **Steps:**
  1. As a guest, navigate to a product page.
  2. Inspect the page HTML/network for `unit_cost`, `cost_price`, or `avg_cost`.
- **Expected UI Result:** No cost data is present in the rendered page or API responses.
- **Expected Backend / Data Result:** `quoteCart` strips `unit_cost`; storefront product queries never select `variant_costs`.
- **Negative Assertions:** Cost price must never be exposed.
- **Cleanup:** None.

---

## 5. RBAC / Permission Matrix

### 5.1 Role → Permission Matrix (from `ShopContext.tsx` `ROLE_MATRIX` and `nav.ts`)

| Permission | Owner | Admin | Manager | Cashier | Accountant | Inventory | Staff |
| --- | --- | --- | --- | --- | --- | --- | --- |
| pos | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| orders | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| products | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| inventory | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| purchasing | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| accounting | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| storefront | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| ai | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| settings | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

> **Server-side guards** (`guards.server.ts`): `MANAGERS = [owner, admin, manager]`, `ADMINS = [owner, admin]`, `INVENTORY_ROLES = [owner, admin, manager, inventory]`, `FINANCE_ROLES = [owner, admin, manager, accountant]`, `SELLING_ROLES = [owner, admin, manager, cashier]`. Every mutating server function re-checks the role server-side.

### RBAC-001 — Cashier can open POS
- **Area:** RBAC
- **Title:** Cashier has POS access
- **Priority:** P0
- **Test Type:** Permission
- **Suitable for Playwright?:** Yes
- **Roles:** Cashier
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as cashier.
  2. Navigate to `/admin/pos`.
- **Expected UI Result:** POS workspace renders.
- **Expected Backend / Data Result:** `posCheckout` allows `SELLING_ROLES`.
- **Negative Assertions:** None.
- **Cleanup:** None.

### RBAC-002 — Accountant cannot open POS
- **Area:** RBAC
- **Title:** Accountant is denied POS access
- **Priority:** P0
- **Test Type:** Permission, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Accountant
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as accountant.
  2. Navigate to `/admin/pos`.
- **Expected UI Result:** POS nav item is hidden; direct navigation may render the shell but POS actions are denied server-side.
- **Expected Backend / Data Result:** `posCheckout` throws `Your role does not allow this action`.
- **Negative Assertions:** Accountant must not complete a sale.
- **Cleanup:** None.

### RBAC-003 — Staff cannot open POS
- **Area:** RBAC
- **Title:** Staff is denied POS access
- **Priority:** P0
- **Test Type:** Permission, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Staff
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as staff.
  2. Navigate to `/admin/pos`.
- **Expected UI Result:** POS nav item is hidden.
- **Expected Backend / Data Result:** `posCheckout` throws.
- **Negative Assertions:** Staff must not complete a sale.
- **Cleanup:** None.

### RBAC-004 — Inventory role can manage products and inventory but not accounting
- **Area:** RBAC
- **Title:** Inventory role access matrix
- **Priority:** P1
- **Test Type:** Permission
- **Suitable for Playwright?:** Yes
- **Roles:** Inventory
- **Steps:**
  1. Authenticate as inventory role.
  2. Verify `/admin/products`, `/admin/inventory`, `/admin/purchasing` are accessible.
  3. Verify `/admin/accounting` nav is hidden.
- **Expected UI Result:** Products/Inventory/Purchasing nav visible; Accounting nav hidden.
- **Expected Backend / Data Result:** `FINANCE_ROLES` excludes inventory; accounting server functions throw.
- **Negative Assertions:** Inventory role must not access accounting reports.
- **Cleanup:** None.

### RBAC-005 — Manager cannot access settings/team/audit
- **Area:** RBAC
- **Title:** Manager is denied administration
- **Priority:** P0
- **Test Type:** Permission, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Manager
- **Route:** `/admin/settings`, `/admin/team`, `/admin/audit`
- **Steps:**
  1. Authenticate as manager.
  2. Navigate to `/admin/settings`, `/admin/team`, `/admin/audit`.
- **Expected UI Result:** Administration nav group is hidden; direct navigation shows `Access restricted` empty state.
- **Expected Backend / Data Result:** Server functions require `ADMINS`; manager is denied.
- **Negative Assertions:** Manager must not change settings or manage team.
- **Cleanup:** None.

### RBAC-006 — Cashier cannot manage products
- **Area:** RBAC
- **Title:** Cashier is denied catalog management
- **Priority:** P1
- **Test Type:** Permission, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Cashier
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as cashier.
  2. Navigate to `/admin/products`.
- **Expected UI Result:** Catalog nav group is hidden.
- **Expected Backend / Data Result:** Product mutation server functions require `MANAGERS`/`INVENTORY_ROLES`; cashier denied.
- **Negative Assertions:** Cashier must not create/edit products.
- **Cleanup:** None.

### RBAC-007 — Staff can view orders but not record payments
- **Area:** RBAC
- **Title:** Staff order access is read-only
- **Priority:** P1
- **Test Type:** Permission, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Staff
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as staff.
  2. Navigate to `/admin/orders`.
  3. Attempt to record a payment on an unpaid order.
- **Expected UI Result:** Orders list visible; payment recording denied.
- **Expected Backend / Data Result:** `recordSalePayment` requires `[owner, admin, manager, accountant]`; staff denied.
- **Negative Assertions:** Staff must not record payments.
- **Cleanup:** None.

### RBAC-008 — Cashier cannot record payments on credit sales
- **Area:** RBAC
- **Title:** Cashier is denied payment recording
- **Priority:** P1
- **Test Type:** Permission, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Cashier
- **Route:** `/admin/sales`
- **Steps:**
  1. Authenticate as cashier.
  2. Navigate to `/admin/sales`.
  3. Attempt to record a payment.
- **Expected UI Result:** Payment recording denied.
- **Expected Backend / Data Result:** `recordSalePayment` requires finance roles; cashier denied.
- **Negative Assertions:** Cashier must not record payments.
- **Cleanup:** None.

### RBAC-009 — Only owner/admin can access the Data Manager
- **Area:** RBAC
- **Title:** Data Manager is owner/admin only
- **Priority:** P0
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Manager, Cashier, Accountant, Inventory, Staff
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as a non-owner/admin role.
  2. Navigate to `/admin/data-manager`.
- **Expected UI Result:** Data Manager nav hidden; direct access denied.
- **Expected Backend / Data Result:** `dmTablesFn` requires `ADMINS`; throws for other roles.
- **Negative Assertions:** Non-admin roles must not run bulk operations.
- **Cleanup:** None.

### RBAC-010 — Only owner can run Data Manager Reset
- **Area:** RBAC
- **Title:** Reset is owner-only
- **Priority:** P0
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Admin
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as admin (not owner).
  2. Navigate to `/admin/data-manager`.
  3. Attempt a Reset.
- **Expected UI Result:** Reset denied.
- **Expected Backend / Data Result:** `dmResetFn` requires `["owner"]`; admin denied.
- **Negative Assertions:** Admin must not reset data.
- **Cleanup:** None.

### RBAC-011 — Only owner/admin can manage custom domains
- **Area:** RBAC
- **Title:** Domain management is owner/admin only
- **Priority:** P1
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Manager
- **Route:** `/admin/storefront` (Domains tab)
- **Steps:**
  1. Authenticate as manager.
  2. Navigate to `/admin/storefront` → Domains tab.
  3. Attempt to add a domain.
- **Expected UI Result:** Domain actions denied.
- **Expected Backend / Data Result:** `addDomainFn` requires `ADMINS`; manager denied.
- **Negative Assertions:** Manager must not add/remove domains.
- **Cleanup:** None.

### RBAC-012 — Only owner/admin can view audit log
- **Area:** RBAC
- **Title:** Audit log is owner/admin only
- **Priority:** P1
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Manager
- **Route:** `/admin/audit`
- **Steps:**
  1. Authenticate as manager.
  2. Navigate to `/admin/audit`.
- **Expected UI Result:** `Access restricted` empty state.
- **Expected Backend / Data Result:** `getAuditLogs` requires `ADMINS`.
- **Negative Assertions:** Manager must not view audit logs.
- **Cleanup:** None.

### RBAC-013 — Only owner/admin can manage team
- **Area:** RBAC
- **Title:** Team management is owner/admin only
- **Priority:** P1
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Manager
- **Route:** `/admin/team`
- **Steps:**
  1. Authenticate as manager.
  2. Navigate to `/admin/team`.
- **Expected UI Result:** `Access restricted` empty state.
- **Expected Backend / Data Result:** `inviteStaffMember` requires `ADMINS`.
- **Negative Assertions:** Manager must not invite/revoke staff.
- **Cleanup:** None.

### RBAC-014 — Cashier cannot open a cash session on an unassigned register
- **Area:** RBAC
- **Title:** Register assignment gates cash session opening
- **Priority:** P1
- **Test Type:** Permission, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Cashier
- **Preconditions:** A register exists; cashier is NOT assigned to it.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as cashier.
  2. Open POS.
  3. Attempt to open a cash session on an unassigned register.
- **Expected UI Result:** Error `You are not assigned to this register`.
- **Expected Backend / Data Result:** `openCashSession` checks `register_users` for non-managers.
- **Negative Assertions:** Cashier must not open a session on an unassigned register.
- **Cleanup:** None.

### RBAC-015 — Cashier can open a cash session on an assigned register
- **Area:** RBAC
- **Title:** Assigned cashier opens a session
- **Priority:** P1
- **Test Type:** Permission, Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Cashier
- **Preconditions:** A register exists; cashier IS assigned to it.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as cashier.
  2. Open POS.
  3. Open a cash session on the assigned register.
- **Expected UI Result:** Session opens.
- **Expected Backend / Data Result:** `cash_sessions` row created with status `open`.
- **Cleanup:** Close the session.

### RBAC-016 — Only manager+ can approve stock counts
- **Area:** RBAC
- **Title:** Stock count approval is manager-gated
- **Priority:** P1
- **Test Type:** Permission, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Inventory
- **Route:** `/admin/stock-counts`
- **Steps:**
  1. Authenticate as inventory role.
  2. Navigate to `/admin/stock-counts`.
  3. Attempt to approve a count.
- **Expected UI Result:** Approve action hidden/denied.
- **Expected Backend / Data Result:** `approveStockCountFn` requires `MANAGERS`.
- **Negative Assertions:** Inventory role must not approve counts.
- **Cleanup:** None.

### RBAC-017 — Only manager+ can process refunds
- **Area:** RBAC
- **Title:** Refund is manager-gated
- **Priority:** P0
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Cashier
- **Route:** `/admin/orders` (return modal)
- **Steps:**
  1. Authenticate as cashier.
  2. Navigate to `/admin/orders`.
  3. Attempt to process a return/refund.
- **Expected UI Result:** Return action denied.
- **Expected Backend / Data Result:** `posRefund` requires `MANAGERS`; cashier denied.
- **Negative Assertions:** Cashier must not process refunds.
- **Cleanup:** None.

### RBAC-018 — Only finance roles can view sales reports (cost/margin)
- **Area:** RBAC
- **Title:** Sales reports are finance-gated
- **Priority:** P1
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Staff, Cashier
- **Route:** `/admin/sales-reports`
- **Steps:**
  1. Authenticate as staff.
  2. Navigate to `/admin/sales-reports`.
- **Expected UI Result:** Report data denied (cost/margin exposed).
- **Expected Backend / Data Result:** `getSalesOverviewReport` requires `FINANCE_ROLES`.
- **Negative Assertions:** Staff/cashier must not see cost/margin reports.
- **Cleanup:** None.

### RBAC-019 — Only inventory/finance roles can view inventory reports (valuation)
- **Area:** RBAC
- **Title:** Inventory reports are inventory/finance-gated
- **Priority:** P1
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Staff, Cashier
- **Route:** `/admin/inventory-reports`
- **Steps:**
  1. Authenticate as staff.
  2. Navigate to `/admin/inventory-reports`.
- **Expected UI Result:** Report data denied (valuation exposed).
- **Expected Backend / Data Result:** `getInventoryValuationReport` requires inventory/finance roles.
- **Negative Assertions:** Staff/cashier must not see valuation.
- **Cleanup:** None.

### RBAC-020 — Only owner/admin can change shop settings
- **Area:** RBAC
- **Title:** Shop settings are owner/admin-gated
- **Priority:** P1
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Manager
- **Route:** `/admin/settings`
- **Steps:**
  1. Authenticate as manager.
  2. Navigate to `/admin/settings`.
  3. Attempt to edit a business/tax section.
- **Expected UI Result:** Business/Tax/Localization sections hidden or read-only for manager.
- **Expected Backend / Data Result:** `updateShopSection` enforces per-section roles; business/tax require `[owner, admin]`.
- **Negative Assertions:** Manager must not change business/tax settings.
- **Cleanup:** None.

### RBAC-021 — Manager can edit receipt/POS/inventory settings
- **Area:** RBAC
- **Title:** Manager has partial settings access
- **Priority:** P2
- **Test Type:** Permission, Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Manager
- **Route:** `/admin/settings`
- **Steps:**
  1. Authenticate as manager.
  2. Navigate to `/admin/settings`.
  3. Edit the Receipts or Inventory section.
- **Expected UI Result:** Save succeeds.
- **Expected Backend / Data Result:** `updateShopSection` allows manager for receipt/inventory sections.
- **Cleanup:** Revert the change.

---

## 6. Dashboard Test Cases

### DASH-001 — Dashboard loads with KPIs
- **Area:** Dashboard
- **Title:** Control Center renders KPI cards
- **Priority:** P1
- **Test Type:** Smoke
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin`
- **Selectors / UI Targets:** heading `Control Center`, StatCard labels `Today's sales`, `Today's orders`, `Gross profit today`, `Gross margin`, `Inventory value`, `Low stock items`, `Receivables`, `Payables`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin`.
- **Expected UI Result:** All 8 KPI cards render with values.
- **Expected Backend / Data Result:** `getDashboardOverview` returns KPIs.
- **Negative Assertions:** No "dancing" layout; skeleton is stable.
- **Cleanup:** None.

### DASH-002 — Dashboard empty state
- **Area:** Dashboard
- **Title:** Empty shop shows zero KPIs
- **Priority:** P2
- **Test Type:** Smoke
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A shop with no sales.
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as owner of an empty shop.
  2. Navigate to `/admin`.
- **Expected UI Result:** KPIs show `₨0.00` / `0`; no errors.
- **Expected Backend / Data Result:** `getDashboardOverview` returns zeros.
- **Cleanup:** None.

### DASH-003 — Dashboard loading skeleton is stable (no layout shift)
- **Area:** Dashboard
- **Title:** Dashboard skeleton mirrors the real layout
- **Priority:** P2
- **Test Type:** Regression, UX
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin`.
  3. Measure the layout during load and after load.
- **Expected UI Result:** No severe layout shift; skeleton dimensions match the real cards.
- **Expected Backend / Data Result:** None.
- **Negative Assertions:** No "dancing" cards.
- **Cleanup:** None.

### DASH-004 — Dashboard branch switch
- **Area:** Dashboard
- **Title:** Branch switcher updates dashboard scope
- **Priority:** P2
- **Test Type:** Integration
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** Shop has multiple branches.
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin`.
  3. Switch branch in the header.
- **Expected UI Result:** Dashboard data reloads for the selected branch.
- **Expected Backend / Data Result:** Branch-scoped queries.
- **Cleanup:** None.

### DASH-005 — Dashboard shop switch
- **Area:** Dashboard
- **Title:** Shop switcher updates dashboard
- **Priority:** P1
- **Test Type:** Integration
- **Suitable for Playwright?:** Yes
- **Roles:** Owner (multi-shop)
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as a multi-shop owner.
  2. Navigate to `/admin`.
  3. Switch shop.
- **Expected UI Result:** Dashboard reloads for the new shop.
- **Expected Backend / Data Result:** `getDashboardOverview` refetches with new `shop_id`.
- **Negative Assertions:** Old shop data must not remain.
- **Cleanup:** None.

### DASH-006 — Dashboard navigation cards link correctly
- **Area:** Dashboard
- **Title:** KPI cards navigate to the right pages
- **Priority:** P2
- **Test Type:** Smoke
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin`.
  3. Click `Today's sales` card.
- **Expected UI Result:** Navigates to `/admin/orders`.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### DASH-007 — Dashboard AI briefing
- **Area:** Dashboard
- **Title:** AI briefing button generates a summary
- **Priority:** P2
- **Test Type:** Smoke
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Owner
- **Route:** `/admin`
- **Selectors / UI Targets:** button `Today's briefing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin`.
  3. Click `Today's briefing`.
- **Expected UI Result:** A briefing text appears.
- **Expected Backend / Data Result:** `getBriefing` returns a summary.
- **Cleanup:** None.

### DASH-008 — Dashboard alert centre
- **Area:** Dashboard
- **Title:** Alerts render and can be dismissed
- **Priority:** P2
- **Test Type:** Smoke
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin`.
  3. Verify the alert centre renders alerts.
- **Expected UI Result:** Alerts visible; dismiss works.
- **Expected Backend / Data Result:** `listAlerts` returns alerts.
- **Cleanup:** None.

---

## 7. Product / Catalog Test Cases

### PRODUCT-001 — Create a simple product
- **Area:** Catalog
- **Title:** Owner creates a product with a variant
- **Priority:** P0
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** Shop exists; a category and brand exist.
- **Test Data:** Product `QA Perfume 100ml`, SKU `QA-PERF-100`, price `100`, cost `60`, tax `0%`, opening stock `10`, category `QA Category`, brand `QA Brand`.
- **Route:** `/admin/products`
- **Selectors / UI Targets:** button `New product`, field `Product Name`, field `SKU`, field `Price`, field `Cost`, button `Save`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Click `New product`.
  4. Fill product name, SKU, price, cost, opening stock.
  5. Save.
- **Expected UI Result:** Product appears in the catalog table.
- **Expected Backend / Data Result:**
  - `products` row created (status `active`, `is_online` true)
  - `product_variants` row created (SKU, price)
  - `variant_costs` row created (cost)
  - `inventory_balances` row created (opening stock 10)
  - `inventory_movements` row created (movement_type `opening`)
  - `document_sequences` product number incremented
- **Side Effects:** Audit log entry `product.*`.
- **Negative Assertions:** No duplicate SKU.
- **Cleanup:** Delete the QA product (cascade variants/costs/balances).
- **Notes for Playwright:** Verify DB rows after save.

### PRODUCT-002 — Duplicate SKU rejected
- **Area:** Catalog
- **Title:** Creating a product with an existing SKU fails
- **Priority:** P1
- **Test Type:** Negative, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A variant with SKU `QA-PERF-100` exists.
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Create a product with SKU `QA-PERF-100`.
- **Expected UI Result:** Error about duplicate SKU.
- **Expected Backend / Data Result:** Unique constraint on `(shop_id, sku)` rejects.
- **Negative Assertions:** No duplicate SKU.
- **Cleanup:** None.

### PRODUCT-003 — Edit a product
- **Area:** Catalog
- **Title:** Owner edits product name/price
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A QA product exists.
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Open the QA product and edit its name/price.
  4. Save.
- **Expected UI Result:** Updated values shown.
- **Expected Backend / Data Result:** `products`/`product_variants` updated; `price_history` row created.
- **Negative Assertions:** None.
- **Cleanup:** None.

### PRODUCT-004 — Archive/deactivate a product
- **Area:** Catalog
- **Title:** Owner archives a product
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A QA product exists.
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Toggle the product status to `draft`/archived.
- **Expected UI Result:** Product status changes.
- **Expected Backend / Data Result:** `products.status` updated.
- **Negative Assertions:** Archived product must not appear in POS/storefront.
- **Cleanup:** Reactivate.

### PRODUCT-005 — "Available for sale" toggle hides product from POS and storefront
- **Area:** Catalog
- **Title:** Internal-only product is hidden from POS and storefront
- **Priority:** P0
- **Test Type:** Integration, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A product with `is_online = false` exists.
- **Route:** `/admin/products`, `/admin/pos`, `/store/<slug>/shop`
- **Steps:**
  1. Authenticate as owner.
  2. Set a QA product's `is_online` to false.
  3. Open POS and search for it.
  4. Open the storefront and search for it.
- **Expected UI Result:** Product is hidden from POS product search and storefront.
- **Expected Backend / Data Result:** `posCatalog` and `listStorefrontProducts` exclude `is_online = false` products.
- **Negative Assertions:** Internal product must not appear in POS/storefront.
- **Cleanup:** Reset `is_online` to true.
- **Notes for Playwright:** The `is_online` filter is enforced by RLS on the anon client and by the POS query.

### PRODUCT-006 — Create a multi-variant product
- **Area:** Catalog
- **Title:** Variant builder creates a variant matrix
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** Attributes (e.g. Size, Colour) exist.
- **Test Data:** Product `QA T-Shirt`, attributes `Size: [S, M]`, `Colour: [Red, Blue]` → 4 variants.
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Click `New product`.
  4. Use the variant builder to create a 2×2 matrix.
  5. Save.
- **Expected UI Result:** 4 variants created.
- **Expected Backend / Data Result:** 4 `product_variants` rows; 4 `variant_costs` rows.
- **Negative Assertions:** No duplicate variant combinations.
- **Cleanup:** Delete the QA product.

### PRODUCT-007 — Duplicate barcode rejected
- **Area:** Catalog
- **Title:** Attaching an in-use barcode fails
- **Priority:** P1
- **Test Type:** Negative, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A barcode is already assigned to a variant.
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Attempt to attach an existing barcode to another variant.
- **Expected UI Result:** Error `Barcode X is already assigned to another variant in this shop`.
- **Expected Backend / Data Result:** `attachManualBarcode` checks `codeInUse`.
- **Negative Assertions:** No duplicate barcode.
- **Cleanup:** None.

### PRODUCT-008 — EAN-13 validation
- **Area:** Catalog
- **Title:** Invalid EAN-13 barcode rejected
- **Priority:** P2
- **Test Type:** Validation, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Attach an EAN-13 barcode with a wrong check digit.
- **Expected UI Result:** Error `Not a valid EAN-13 barcode (check digit mismatch)`.
- **Expected Backend / Data Result:** `isValidEan13` fails.
- **Cleanup:** None.

### PRODUCT-009 — Barcode generation
- **Area:** Catalog
- **Title:** Generate a barcode for a variant
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Generate a barcode for a variant without one.
- **Expected UI Result:** A barcode is generated and shown.
- **Expected Backend / Data Result:** `barcodes` row created (source `generated`).
- **Cleanup:** None.

### PRODUCT-010 — Product search and filtering
- **Area:** Catalog
- **Title:** Search by name/SKU/barcode and filter by status/category/brand
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Selectors / UI Targets:** textbox `Search product, SKU, tag or barcode…`, buttons `All statuses`, `All categories`, `All brands`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Search for a QA SKU.
  4. Apply status/category/brand filters.
- **Expected UI Result:** Matching products shown; filters work.
- **Expected Backend / Data Result:** Client-side filtering (products table is client-filtered per handoff).
- **Cleanup:** None.
- **Notes for Playwright:** Products table is client-side filtered (not server-paginated) — see handoff "Deferred — High Impact".

### PRODUCT-011 — CSV import of products
- **Area:** Catalog
- **Title:** Import products from CSV with mapping and validation
- **Priority:** P1
- **Test Type:** Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/import`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/import`.
  3. Upload a CSV with QA products.
  4. Map columns, validate, commit.
- **Expected UI Result:** Products created; validation summary shown.
- **Expected Backend / Data Result:** `commitImport` creates products/variants.
- **Negative Assertions:** Invalid rows are skipped with errors.
- **Cleanup:** Delete imported QA products.

### PRODUCT-012 — Import with invalid rows
- **Area:** Catalog
- **Title:** Import validation flags missing/invalid data
- **Priority:** P2
- **Test Type:** Negative, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/import`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/import`.
  3. Upload a CSV with a row missing SKU/price.
- **Expected UI Result:** Validation shows errors for the bad rows.
- **Expected Backend / Data Result:** `validateImport` returns error rows.
- **Negative Assertions:** Bad rows are not committed.
- **Cleanup:** None.

### PRODUCT-013 — Cost price is not exposed to non-finance roles
- **Area:** Catalog
- **Title:** Cost price hidden from cashier/staff
- **Priority:** P0
- **Test Type:** Security
- **Suitable for Playwright?:** Yes
- **Roles:** Cashier
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as cashier.
  2. Navigate to `/admin/products` (if accessible) or attempt to view product detail.
- **Expected UI Result:** Cost price not shown.
- **Expected Backend / Data Result:** Product detail queries that expose cost require finance roles.
- **Negative Assertions:** Cost must not leak to cashier.
- **Cleanup:** None.

### PRODUCT-014 — Zero/negative price validation
- **Area:** Catalog
- **Title:** Negative price rejected
- **Priority:** P2
- **Test Type:** Validation, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Attempt to set a negative price.
- **Expected UI Result:** Validation error.
- **Expected Backend / Data Result:** Price constraint rejects negative.
- **Cleanup:** None.

### PRODUCT-015 — Barcode label printing
- **Area:** Catalog
- **Title:** Print barcode labels for selected variants
- **Priority:** P2
- **Test Type:** Smoke
- **Suitable for Playwright?:** Partial (print dialog)
- **Roles:** Owner
- **Route:** `/admin/labels`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/labels`.
  3. Select variants, set copies, click `Print`.
- **Expected UI Result:** Print sheet renders; `window.print()` invoked.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

---

## 8. POS Test Cases (P0 — very detailed)

### 8.1 Register / Session

### POS-001 — POS loads with register and no open session
- **Area:** POS
- **Title:** POS workspace renders with register picker
- **Priority:** P0
- **Test Type:** Smoke
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** banner with shop name, `Register 1`, `No open session`, buttons `Register`, `Open session`, `Recent sales`, `Display`, `Registers`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
- **Expected UI Result:** POS shell renders; register picker shows `Register 1`; `No open session` badge.
- **Expected Backend / Data Result:** `listRegisters` returns registers.
- **Cleanup:** None.

### POS-002 — Open a cash session with float
- **Area:** POS
- **Title:** Owner opens a cash session with an opening float
- **Priority:** P0
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A register exists; no open session.
- **Test Data:** Opening float `1000`.
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** textbox `Opening float`, button `Open session`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Enter opening float `1000`.
  4. Click `Open session`.
- **Expected UI Result:** Session opens; badge changes from `No open session` to open.
- **Expected Backend / Data Result:**
  - `cash_sessions` row created (status `open`, opening_balance `1000`)
  - `cash_movements` row created (movement_type `opening`, amount `1000`)
  - Audit log `cash_session.open`
- **Negative Assertions:** No duplicate open session.
- **Cleanup:** Close the session.
- **Notes for Playwright:** Serial — only one open session per register.

### POS-003 — Duplicate open session rejected
- **Area:** POS
- **Title:** Opening a session on an already-open register fails
- **Priority:** P1
- **Test Type:** Negative, Concurrency
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A register already has an open session.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Attempt to open a session on the already-open register.
- **Expected UI Result:** Error `This register already has an open session`.
- **Expected Backend / Data Result:** `openCashSession` checks for existing open session.
- **Negative Assertions:** No second open session.
- **Cleanup:** Close the session.

### POS-004 — Negative opening float rejected
- **Area:** POS
- **Title:** Negative opening balance rejected
- **Priority:** P2
- **Test Type:** Validation, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Enter a negative opening float.
- **Expected UI Result:** Error `Opening balance cannot be negative`.
- **Expected Backend / Data Result:** `openCashSession` rejects negative.
- **Cleanup:** None.

### POS-005 — Cash in / cash out movements
- **Area:** POS
- **Title:** Record cash in/out during a session
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** An open cash session exists.
- **Route:** `/admin/registers` (session detail)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/registers`.
  3. Open the session detail.
  4. Record a cash-in of `500`.
- **Expected UI Result:** Movement recorded.
- **Expected Backend / Data Result:** `cash_movements` row created (movement_type `cash_in`).
- **Negative Assertions:** Amount must be > 0.
- **Cleanup:** None.

### POS-006 — Close a cash session with counted cash and variance
- **Area:** POS
- **Title:** Close session with counted balance and variance
- **Priority:** P0
- **Test Type:** Happy Path, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** An open cash session exists with known sales.
- **Route:** `/admin/registers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/registers`.
  3. Open the session detail.
  4. Enter counted balance and close.
- **Expected UI Result:** Session closes; variance shown.
- **Expected Backend / Data Result:**
  - `cash_sessions.status` → `closed`
  - `closing_balance`, `counted_balance`, `discrepancy` set
  - Audit log `cash_session.close`
- **Negative Assertions:** Variance computed correctly.
- **Cleanup:** None.

### 8.2 Product Discovery

### POS-007 — Scan a barcode adds item to cart
- **Area:** POS
- **Title:** Barcode scan adds the product
- **Priority:** P0
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A product with a barcode exists.
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** textbox `Scan barcode or SKU, then Enter`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Type a barcode into the scan field and press Enter.
- **Expected UI Result:** The product is added to the cart.
- **Expected Backend / Data Result:** `lookupVariantByBarcode` resolves the variant.
- **Cleanup:** Clear the cart.

### POS-008 — Fuzzy search finds a product
- **Area:** POS
- **Title:** Search by name/SKU/brand/category
- **Priority:** P0
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** textbox `Search name, SKU, brand or category (F2)`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Type a partial product name.
- **Expected UI Result:** Matching products shown in the grid.
- **Expected Backend / Data Result:** Client-side fuzzy search over the catalog.
- **Cleanup:** None.

### POS-009 — Category filter in POS
- **Area:** POS
- **Title:** Category chips filter the product grid
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** category chips (e.g. `All items`, `Bags`).
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Click a category chip.
- **Expected UI Result:** Product grid filters to that category.
- **Cleanup:** None.

### POS-010 — Unavailable-for-sale product hidden from POS
- **Area:** POS
- **Title:** Internal-only product not in POS catalog
- **Priority:** P0
- **Test Type:** Security, Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A product with `is_online = false` exists.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Search for the internal product.
- **Expected UI Result:** Product not found.
- **Expected Backend / Data Result:** `posCatalog` excludes `is_online = false`.
- **Negative Assertions:** Internal product must not be sellable at POS.
- **Cleanup:** None.

### 8.3 Cart

### POS-011 — Add item to cart and update total
- **Area:** POS
- **Title:** Tapping a product adds it and updates the total
- **Priority:** P0
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** product tiles `[data-pos="product-tile"]`, cart `[data-pos="cart"]`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Tap a product tile.
- **Expected UI Result:** Cart shows the line; total updates.
- **Expected Backend / Data Result:** None (client-side cart).
- **Cleanup:** Clear the cart.

### POS-012 — Repeated add increments quantity
- **Area:** POS
- **Title:** Tapping the same product twice increments the line
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Tap the same product twice.
- **Expected UI Result:** One line with quantity 2.
- **Expected Backend / Data Result:** None.
- **Cleanup:** Clear the cart.

### POS-013 — Increase/decrease quantity
- **Area:** POS
- **Title:** Quantity steppers work
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** cart `Increase quantity` / `Decrease quantity` buttons.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product.
  4. Increase then decrease quantity.
- **Expected UI Result:** Quantity updates; decreasing past 1 removes the line.
- **Expected Backend / Data Result:** None.
- **Cleanup:** Clear the cart.

### POS-014 — Remove a cart line
- **Area:** POS
- **Title:** Remove item from cart
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product.
  4. Remove it.
- **Expected UI Result:** Cart is empty again.
- **Cleanup:** None.

### POS-015 — Line discount
- **Area:** POS
- **Title:** Apply a per-line discount
- **Priority:** P1
- **Test Type:** Happy Path, Financial Integrity
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product.
  4. Apply a line discount.
- **Expected UI Result:** Line total reflects the discount.
- **Expected Backend / Data Result:** On checkout, `priceCart` applies the discount server-side.
- **Negative Assertions:** Discount must not exceed the line total (see FIX-001).
- **Cleanup:** Clear the cart.

### POS-016 — Order discount
- **Area:** POS
- **Title:** Apply an order-level discount
- **Priority:** P1
- **Test Type:** Happy Path, Financial Integrity
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add products.
  4. Apply an order discount.
- **Expected UI Result:** Order total reflects the discount.
- **Expected Backend / Data Result:** On checkout, `discount_total` reflects the order discount.
- **Cleanup:** Clear the cart.

### POS-017 — Attach a customer
- **Area:** POS
- **Title:** Attach a customer to the sale
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A customer exists.
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** button `Customer` (F4).
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Open the customer picker (F4).
  4. Select a customer.
- **Expected UI Result:** Customer attached to the cart.
- **Expected Backend / Data Result:** On checkout, `customer_id` is set on the order.
- **Cleanup:** Clear the cart.

### POS-018 — Hold and resume a sale
- **Area:** POS
- **Title:** Hold a sale and resume it
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** button `Hold` (F8).
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add products.
  4. Hold the sale.
  5. Resume it.
- **Expected UI Result:** Held sale is restored.
- **Expected Backend / Data Result:** None (client-side hold).
- **Cleanup:** Clear the cart.

### POS-019 — Cancel cart
- **Area:** POS
- **Title:** Cancel/clear the cart
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Selectors / UI Targets:** `Esc` shortcut.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add products.
  4. Press Esc / cancel.
- **Expected UI Result:** Cart is cleared.
- **Cleanup:** None.

### 8.4 Inventory Restrictions

### POS-020 — Sell within stock
- **Area:** POS
- **Title:** Selling within available stock succeeds
- **Priority:** P0
- **Test Type:** Happy Path, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A variant has 10 units at the location.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add 5 units of the variant.
  4. Complete the sale.
- **Expected UI Result:** Sale completes.
- **Expected Backend / Data Result:** Inventory deducted by 5; balance = 5.
- **Negative Assertions:** No oversell.
- **Cleanup:** Return/restock or adjust.

### POS-021 — Attempt oversell (non-permitted role)
- **Area:** POS
- **Title:** Selling past stock is blocked for non-oversell roles
- **Priority:** P0
- **Test Type:** Negative, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Cashier (not in oversell_roles)
- **Preconditions:** A variant has 1 unit; shop `oversell_roles` excludes cashier.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as cashier.
  2. Navigate to `/admin/pos`.
  3. Add 2 units of the 1-unit variant.
  4. Attempt to complete the sale.
- **Expected UI Result:** Sale blocked (stock error).
- **Expected Backend / Data Result:** `record_pos_sale` enforces stock; throws.
- **Negative Assertions:** No oversell.
- **Cleanup:** None.

### POS-022 — Permitted role oversell
- **Area:** POS
- **Title:** Owner/manager can oversell (negative stock)
- **Priority:** P1
- **Test Type:** Happy Path, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A variant has 1 unit; shop `oversell_roles` includes owner.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add 2 units of the 1-unit variant.
  4. Complete the sale.
- **Expected UI Result:** Sale completes; balance goes negative.
- **Expected Backend / Data Result:** `allow_oversell` true for owner; balance = -1.
- **Cleanup:** Adjust stock back.

### 8.5 Payment

### POS-023 — Cash payment with change
- **Area:** POS
- **Title:** Cash tender with change due
- **Priority:** P0
- **Test Type:** Happy Path, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** An open cash session exists; a product in cart.
- **Test Data:** Order total `100`, cash tendered `150`.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product totaling 100.
  4. Open payment (F9).
  5. Enter cash `150`.
  6. Complete sale.
- **Expected UI Result:** Sale completes; change due `50`.
- **Expected Backend / Data Result:**
  - `orders` row (total 100, paid_total 100, payment_status `paid`)
  - `payments` row (cash, amount 100 — the applied amount after change)
  - `cash_movements` row (sale, amount 100)
  - Inventory deducted once
  - Balanced journal entry
  - `receipts` row with receipt_number
- **Negative Assertions:** Change is not recorded as a payment; inventory deducted once.
- **Cleanup:** Return/restock.
- **Notes for Playwright:** Verify the applied payment is 100 (not 150) — change is returned in cash.

### POS-024 — Split payment (cash + card)
- **Area:** POS
- **Title:** Split payment across methods
- **Priority:** P0
- **Test Type:** Happy Path, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** An open cash session exists.
- **Test Data:** Order total `100`, cash `50` + card `50`.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product totaling 100.
  4. Open payment.
  5. Add cash `50` and card `50`.
  6. Complete sale.
- **Expected UI Result:** Sale completes.
- **Expected Backend / Data Result:** Two `payments` rows (cash 50, card 50); journal debits Cash 50 + Bank 50.
- **Negative Assertions:** Total payments = order total.
- **Cleanup:** Return/restock.

### POS-025 — Under-allocation rejected
- **Area:** POS
- **Title:** Payment less than order total is rejected
- **Priority:** P0
- **Test Type:** Negative, Financial Integrity
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A product in cart.
- **Test Data:** Order total `100`, payment `50`.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product totaling 100.
  4. Enter payment `50`.
  5. Attempt to complete.
- **Expected UI Result:** Error `Payment total 50 is less than the order total 100`.
- **Expected Backend / Data Result:** `recordPosSale` rejects under-payment.
- **Negative Assertions:** No under-paid order.
- **Cleanup:** Clear the cart.

### POS-026 — Non-cash over-payment rejected
- **Area:** POS
- **Title:** Card/bank over-payment rejected (change only in cash)
- **Priority:** P1
- **Test Type:** Negative, Financial Integrity
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A product in cart.
- **Test Data:** Order total `100`, card `150`.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product totaling 100.
  4. Enter card `150`.
  5. Attempt to complete.
- **Expected UI Result:** Error `Non-cash payments (150) exceed the order total 100; change can only be returned in cash`.
- **Expected Backend / Data Result:** `recordPosSale` rejects non-cash over-payment.
- **Negative Assertions:** No non-cash over-payment.
- **Cleanup:** Clear the cart.

### POS-027 — Credit (udhaar) sale
- **Area:** POS
- **Title:** Credit sale with partial/zero payment books to AR
- **Priority:** P0
- **Test Type:** Happy Path, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** Credit sales enabled in settings; a customer with credit limit exists; an open cash session.
- **Test Data:** Order total `100`, credit tender `0`, customer credit limit `500`.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product totaling 100.
  4. Attach a customer.
  5. Select `Credit (pay later)` tender.
  6. Complete sale.
- **Expected UI Result:** Sale completes as unpaid/partial.
- **Expected Backend / Data Result:**
  - `orders` row (payment_status `unpaid`, paid_total 0)
  - Journal debits Accounts Receivable (1150) 100
  - Customer outstanding increases
- **Negative Assertions:** Credit limit not exceeded.
- **Cleanup:** Record payment or return.

### POS-028 — Credit limit exceeded
- **Area:** POS
- **Title:** Credit sale over the customer's limit is rejected
- **Priority:** P0
- **Test Type:** Negative, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A customer with credit limit `100` and existing outstanding `80`.
- **Test Data:** New credit sale `50` (would exceed limit).
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product totaling 50.
  4. Attach the customer.
  5. Select credit tender.
  6. Attempt to complete.
- **Expected UI Result:** Error `Credit limit exceeded...`.
- **Expected Backend / Data Result:** `recordPosSale` rejects.
- **Negative Assertions:** No credit sale over the limit.
- **Cleanup:** None.

### POS-029 — Credit sale requires a customer
- **Area:** POS
- **Title:** Credit sale without a customer is rejected
- **Priority:** P1
- **Test Type:** Negative, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product.
  4. Select credit tender without a customer.
  5. Attempt to complete.
- **Expected UI Result:** Error `Select a customer before making a credit (pay later) sale.`
- **Expected Backend / Data Result:** `posCheckout` rejects.
- **Cleanup:** Clear the cart.

### POS-030 — Duplicate click on Complete Sale
- **Area:** POS
- **Title:** Double-clicking Complete Sale creates one order
- **Priority:** P0
- **Test Type:** Concurrency
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A product in cart; an open cash session.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product.
  4. Open payment.
  5. Click `Complete sale` twice rapidly.
- **Expected UI Result:** Only one order is created; button shows `Processing…` and is disabled.
- **Expected Backend / Data Result:** Exactly one `orders` row; inventory deducted once.
- **Negative Assertions:** No duplicate order; no double inventory deduction.
- **Cleanup:** Return/restock.
- **Notes for Playwright:** The button disables while busy (see features.md "Duplicate async submissions prevented").

### POS-031 — Server recalculates price/tax (frontend price not trusted)
- **Area:** POS
- **Title:** Server-authoritative pricing
- **Priority:** P0
- **Test Type:** Financial Integrity, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A product in cart.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add a product.
  4. Intercept/modify the client price before checkout.
  5. Complete the sale.
- **Expected UI Result:** The order total reflects the server price, not the modified client price.
- **Expected Backend / Data Result:** `priceCart` recomputes from the DB; client price ignored.
- **Negative Assertions:** Client-supplied price must not be trusted.
- **Cleanup:** Return/restock.
- **Notes for Playwright:** This is best verified via API-level test (call `posCheckout` with a tampered price).

### 8.6 Completion Assertions

### POS-032 — Full sale completion assertions
- **Area:** POS
- **Title:** Completed sale creates all dependent records
- **Priority:** P0
- **Test Type:** Financial Integrity, Inventory Integrity, Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A product with cost 60, price 100, tax 0%, stock 10; an open cash session.
- **Test Data:** Sell 2 units for cash 200.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Add 2 units of the product.
  4. Complete the sale for cash 200.
- **Expected Backend / Data Result (verify all):**
  - Exactly one `orders` row (channel `pos`, status `completed`, payment_status `paid`, total 200, paid_total 200)
  - One `order_items` row (quantity 2, unit_price 100, total 200)
  - One `payments` row (cash, 200)
  - Inventory deducted once: balance 8
  - One `inventory_movements` row (movement_type `sale`, quantity -2)
  - One balanced `journal_entries` row (Dr Cash 200, Cr Revenue 200; Dr COGS 120, Cr Inventory 120)
  - One `receipts` row with receipt_number
  - `document_sequences` order/receipt incremented
  - `events` row (SaleCompleted)
- **Negative Assertions:** Inventory deducted exactly once; journal balanced.
- **Cleanup:** Return/restock.
- **Notes for Playwright:** This is the single most important financial integrity test. Verify all DB rows.

### 8.7 Receipt

### POS-033 — Receipt renders with correct totals
- **Area:** POS
- **Title:** Receipt shows correct shop details and totals
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A completed sale exists.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Complete a sale.
  4. View the receipt.
- **Expected UI Result:** Receipt shows shop name, order number, receipt number, line items, totals, payments, change.
- **Expected Backend / Data Result:** `receipts.snapshot` contains the data.
- **Cleanup:** None.

### POS-034 — Receipt reprint
- **Area:** POS
- **Title:** Reprint a previous receipt
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A completed sale exists.
- **Route:** `/admin/pos` (Recent sales)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Open Recent sales.
  4. Reprint a receipt.
- **Expected UI Result:** Receipt reprints.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### 8.8 Keyboard Shortcuts

### POS-035 — Keyboard shortcuts (F2/F4/F6/F8/F9/Esc)
- **Area:** POS
- **Title:** Shortcut keys trigger actions
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes (keyboard)
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Press F2 (search), F4 (customer), F6 (discount), F8 (hold), F9 (pay), Esc (cancel).
- **Expected UI Result:** Each shortcut triggers the corresponding action.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

---

## 9. Offline POS Test Cases

> These require browser context manipulation: `context.setOffline(true)` / `context.setOffline(false)`.

### OFFLINE-001 — App shows offline indicator
- **Area:** Offline POS
- **Title:** POS shows offline status when the network drops
- **Priority:** P1
- **Test Type:** Offline
- **Suitable for Playwright?:** Yes (context.setOffline)
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Set the context offline.
- **Expected UI Result:** Offline badge appears in the session bar.
- **Expected Backend / Data Result:** None.
- **Cleanup:** Restore online.

### OFFLINE-002 — Offline sale is queued
- **Area:** Offline POS
- **Title:** A sale made offline is queued locally
- **Priority:** P0
- **Test Type:** Offline, Data Integrity
- **Suitable for Playwright?:** Yes (context.setOffline)
- **Roles:** Owner
- **Preconditions:** Catalog is cached (visit POS online first).
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos` (online) to cache the catalog.
  3. Set the context offline.
  4. Add a product and complete the sale.
- **Expected UI Result:** Sale is queued; offline badge shows pending count.
- **Expected Backend / Data Result:** A `pending-sales` row is stored in IndexedDB with a `local_uuid`.
- **Negative Assertions:** No order is created server-side yet.
- **Cleanup:** Restore online and sync.

### OFFLINE-003 — Reconnect and sync creates one order
- **Area:** Offline POS
- **Title:** Offline sale syncs without duplication
- **Priority:** P0
- **Test Type:** Offline, Data Integrity, Concurrency
- **Suitable for Playwright?:** Yes (context.setOffline)
- **Roles:** Owner
- **Preconditions:** An offline sale is queued.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Make an offline sale.
  4. Restore online.
  5. Trigger sync.
- **Expected UI Result:** Sale syncs; pending count drops to 0.
- **Expected Backend / Data Result:** Exactly one `orders` row created (deduped by `local_uuid`); inventory deducted once.
- **Negative Assertions:** No duplicate order; no double inventory deduction.
- **Cleanup:** Return/restock.
- **Notes for Playwright:** Verify the `local_uuid` unique constraint prevents duplicates.

### OFFLINE-004 — Multiple pending offline sales sync
- **Area:** Offline POS
- **Title:** Multiple queued sales sync correctly
- **Priority:** P1
- **Test Type:** Offline, Data Integrity
- **Suitable for Playwright?:** Yes (context.setOffline)
- **Roles:** Owner
- **Preconditions:** Catalog cached.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Make 3 offline sales.
  4. Restore online and sync.
- **Expected UI Result:** All 3 sync; pending count 0.
- **Expected Backend / Data Result:** 3 `orders` rows; each with a unique `local_uuid`.
- **Negative Assertions:** No duplicates.
- **Cleanup:** Return/restock.

### OFFLINE-005 — Price variance conflict recorded
- **Area:** Offline POS
- **Title:** Offline sale with changed price records a variance
- **Priority:** P1
- **Test Type:** Offline, Data Integrity
- **Suitable for Playwright?:** Yes (context.setOffline)
- **Roles:** Owner
- **Preconditions:** Catalog cached.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Make an offline sale.
  4. Change the product price (online).
  5. Restore online and sync.
- **Expected UI Result:** A `price_variance` conflict is recorded in `/admin/sync`.
- **Expected Backend / Data Result:** `pos_sync_conflicts` row created (kind `price_variance`).
- **Negative Assertions:** The posted order is not mutated.
- **Cleanup:** Resolve the conflict.

### OFFLINE-006 — Sync conflict console (accept/reject/retry)
- **Area:** Offline POS
- **Title:** Resolve a sync conflict
- **Priority:** P1
- **Test Type:** Offline, Integration
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A sync conflict exists.
- **Route:** `/admin/sync`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sync`.
  3. Accept/reject/retry a conflict.
- **Expected UI Result:** Conflict status updates.
- **Expected Backend / Data Result:** `resolveSyncConflict` updates the conflict.
- **Cleanup:** None.

### OFFLINE-007 — Browser reload while a sale is pending
- **Area:** Offline POS
- **Title:** Pending sale survives a reload
- **Priority:** P2
- **Test Type:** Offline, Data Integrity
- **Suitable for Playwright?:** Yes (context.setOffline)
- **Roles:** Owner
- **Preconditions:** Catalog cached.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Make an offline sale.
  4. Reload the page.
- **Expected UI Result:** The pending sale is still queued (IndexedDB persistence).
- **Expected Backend / Data Result:** `pending-sales` row persists.
- **Cleanup:** Sync.

---

## 10. Returns / Refunds / Exchanges Test Cases

### RETURN-001 — Full return with restock
- **Area:** Returns
- **Title:** Full return restocks inventory and posts a reversing journal
- **Priority:** P0
- **Test Type:** Financial Integrity, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A completed, paid sale exists (1 item, cost 60, price 100).
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Open the sale.
  4. Process a full return with restock.
- **Expected UI Result:** Return recorded; refund shown.
- **Expected Backend / Data Result:**
  - `returns` row created (return_number)
  - `return_items` row (quantity 1, restock true)
  - Inventory restocked: balance +1
  - `inventory_movements` row (movement_type `return`, quantity +1)
  - Balanced reversing journal (Dr Revenue, Cr Cash; Dr Tax Payable; Dr Inventory, Cr COGS)
  - `orders.status` → `refunded`
- **Negative Assertions:** Inventory restocked exactly once; journal balanced.
- **Cleanup:** None.
- **Notes for Playwright:** Verify the reversing journal is balanced (debits = credits).

### RETURN-002 — Partial return
- **Area:** Returns
- **Title:** Partial quantity return
- **Priority:** P1
- **Test Type:** Financial Integrity, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A sale of 3 units exists.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Open the sale.
  4. Return 1 of 3 units.
- **Expected UI Result:** Return of 1 unit recorded.
- **Expected Backend / Data Result:** `return_items` quantity 1; inventory +1; order status stays `completed` (partial).
- **Negative Assertions:** Only 1 unit restocked.
- **Cleanup:** None.

### RETURN-003 — Attempt over-return
- **Area:** Returns
- **Title:** Returning more than sold is capped/rejected
- **Priority:** P0
- **Test Type:** Negative, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A sale of 2 units exists.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Open the sale.
  4. Attempt to return 5 units.
- **Expected UI Result:** Return capped at the remaining returnable quantity (2), or rejected.
- **Expected Backend / Data Result:** `recordReturn` caps `qty = Math.min(item.quantity, remaining)`.
- **Negative Assertions:** No over-return.
- **Cleanup:** None.

### RETURN-004 — Second return after a partial return
- **Area:** Returns
- **Title:** Remaining quantity can be returned later
- **Priority:** P1
- **Test Type:** Happy Path, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A sale of 3 units with 1 already returned.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Open the sale.
  4. Return the remaining 2 units.
- **Expected UI Result:** Return of 2 units recorded.
- **Expected Backend / Data Result:** `alreadyReturned` prevents double-return; remaining 2 returned.
- **Negative Assertions:** No double-return.
- **Cleanup:** None.

### RETURN-005 — Refund on an unpaid order rejected
- **Area:** Returns
- **Title:** Cannot refund an unpaid order
- **Priority:** P0
- **Test Type:** Negative, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** An unpaid (COD/credit) order exists.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Open the unpaid order.
  4. Attempt to refund.
- **Expected UI Result:** Error `This order has not been paid, so there is nothing to refund.`
- **Expected Backend / Data Result:** `recordReturn` rejects when `paid_total <= 0`.
- **Negative Assertions:** No refund on unpaid order.
- **Cleanup:** None.

### RETURN-006 — Non-restock return
- **Area:** Returns
- **Title:** Return without restock
- **Priority:** P2
- **Test Type:** Happy Path, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A completed sale exists.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Open the sale.
  4. Return with restock = false.
- **Expected UI Result:** Return recorded; no restock.
- **Expected Backend / Data Result:** `return_items.restock` false; no inventory movement.
- **Negative Assertions:** Inventory not restocked.
- **Cleanup:** None.

---

## 11. Customers / CRM Test Cases

### CUSTOMER-001 — Create a customer
- **Area:** CRM
- **Title:** Owner creates a customer
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Test Data:** `QA Customer One`, phone `+92 300 0000001`, email `qa.customer1@example.com`.
- **Route:** `/admin/customers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/customers`.
  3. Create a customer.
- **Expected UI Result:** Customer appears in the directory.
- **Expected Backend / Data Result:** `customers` row created.
- **Cleanup:** Delete the QA customer.

### CUSTOMER-002 — Edit a customer
- **Area:** CRM
- **Title:** Owner edits customer CRM fields
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A QA customer exists.
- **Route:** `/admin/customers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/customers`.
  3. Edit the customer's name/phone/notes.
- **Expected UI Result:** Updated values shown.
- **Expected Backend / Data Result:** `updateCustomer` updates the row.
- **Cleanup:** None.

### CUSTOMER-003 — Customer search
- **Area:** CRM
- **Title:** Search customers by name/phone/email
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** QA customers exist.
- **Route:** `/admin/customers`
- **Selectors / UI Targets:** textbox `Search name, phone or email`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/customers`.
  3. Search for a QA customer.
- **Expected UI Result:** Matching customers shown.
- **Expected Backend / Data Result:** `listCustomers` filters by search.
- **Cleanup:** None.

### CUSTOMER-004 — Customer purchase history and metrics
- **Area:** CRM
- **Title:** Customer shows spend, orders, AOV, segment
- **Priority:** P1
- **Test Type:** Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A customer with orders exists.
- **Route:** `/admin/customers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/customers`.
  3. View the customer's metrics.
- **Expected UI Result:** Orders, spend, AOV, last order, segment shown.
- **Expected Backend / Data Result:** `listCustomers` aggregates order metrics.
- **Cleanup:** None.

### CUSTOMER-005 — Customer segmentation
- **Area:** CRM
- **Title:** Segments (VIP/returning/new/at-risk/dormant) computed
- **Priority:** P2
- **Test Type:** Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** Customers with varying order history.
- **Route:** `/admin/customers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/customers`.
  3. Verify segment distribution.
- **Expected UI Result:** Segments shown.
- **Expected Backend / Data Result:** `segmentOf` computes segments.
- **Cleanup:** None.

### CUSTOMER-006 — Guest order creates a customer
- **Area:** CRM
- **Title:** Guest checkout creates a customer record
- **Priority:** P1
- **Test Type:** Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** A product in stock.
- **Route:** `/store/<slug>/checkout`
- **Steps:**
  1. As a guest, add a product to cart.
  2. Checkout with a new email.
  3. Place the order.
- **Expected UI Result:** Order placed.
- **Expected Backend / Data Result:** A `customers` row is created (or reused if email matches a guest record).
- **Negative Assertions:** A guest's order is not attached to a registered account with the same email.
- **Cleanup:** None.

### CUSTOMER-007 — Customer-linked POS sale
- **Area:** CRM
- **Title:** POS sale with an attached customer updates their metrics
- **Priority:** P1
- **Test Type:** Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A customer exists.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Attach a customer.
  4. Complete a sale.
- **Expected UI Result:** Customer's order count/spend increases.
- **Expected Backend / Data Result:** `orders.customer_id` set.
- **Cleanup:** Return/restock.

### CUSTOMER-008 — Duplicate email/phone behaviour
- **Area:** CRM
- **Title:** Duplicate customer email handling
- **Priority:** P2
- **Test Type:** Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A customer with a given email exists.
- **Route:** `/admin/customers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/customers`.
  3. Attempt to create a customer with a duplicate email.
- **Expected UI Result:** Duplicate allowed or warned (no unique constraint on email per shop).
- **Expected Backend / Data Result:** Depends on schema; document actual behaviour.
- **Cleanup:** None.

---

## 12. Inventory Test Cases

### INV-001 — Opening stock creates a balance
- **Area:** Inventory
- **Title:** Product opening stock creates an inventory balance
- **Priority:** P0
- **Test Type:** Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A product with opening stock 10.
- **Route:** `/admin/inventory`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/inventory`.
  3. Verify the product's balance.
- **Expected UI Result:** Balance shows 10.
- **Expected Backend / Data Result:** `inventory_balances` row (quantity 10); `inventory_movements` (opening).
- **Cleanup:** None.

### INV-002 — Manual adjustment increase
- **Area:** Inventory
- **Title:** Post a positive stock adjustment
- **Priority:** P1
- **Test Type:** Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A variant exists.
- **Route:** `/admin/inventory`
- **Selectors / UI Targets:** field `Product`, field `Quantity (+/-)`, field `Reason`, button `Post adjustment`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/inventory`.
  3. Select a variant, enter quantity +5, reason.
  4. Click `Post adjustment`.
- **Expected UI Result:** Balance increases by 5.
- **Expected Backend / Data Result:** `inventory_movements` (adjustment, +5); balance updated.
- **Cleanup:** Reverse the adjustment.

### INV-003 — Manual adjustment decrease
- **Area:** Inventory
- **Title:** Post a negative stock adjustment
- **Priority:** P1
- **Test Type:** Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A variant with stock.
- **Route:** `/admin/inventory`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/inventory`.
  3. Select a variant, enter quantity -3.
  4. Click `Post adjustment`.
- **Expected UI Result:** Balance decreases by 3.
- **Expected Backend / Data Result:** `inventory_movements` (adjustment, -3); balance updated.
- **Cleanup:** Reverse the adjustment.

### INV-004 — Inventory movement log
- **Area:** Inventory
- **Title:** Movement log lists all movements with filters
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/inventory`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/inventory`.
  3. View the movement log.
  4. Filter by type/date.
- **Expected UI Result:** Movements listed; filters work.
- **Expected Backend / Data Result:** Server-side paginated movements query.
- **Cleanup:** None.

### INV-005 — Stock transfer lifecycle
- **Area:** Inventory
- **Title:** Create → request → approve → ship → receive a transfer
- **Priority:** P0
- **Test Type:** Inventory Integrity, Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** Two locations exist; stock at the source.
- **Route:** `/admin/transfers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/transfers`.
  3. Create a transfer (draft).
  4. Request it.
  5. Approve it.
  6. Ship it.
  7. Receive it.
- **Expected UI Result:** Transfer progresses through statuses.
- **Expected Backend / Data Result:**
  - `inventory_transfers` status transitions
  - On ship: source balance decreases
  - On receive: destination balance increases
  - `inventory_movements` (transfer_out, transfer_in)
- **Negative Assertions:** Stock moves exactly once; same-location transfer rejected.
- **Cleanup:** Reverse the transfer.

### INV-006 — Same-location transfer rejected
- **Area:** Inventory
- **Title:** Transfer with same source/destination fails
- **Priority:** P1
- **Test Type:** Negative, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/transfers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/transfers`.
  3. Create a transfer with the same from/to location.
- **Expected UI Result:** Error `Source and destination locations must differ`.
- **Expected Backend / Data Result:** `createTransfer` rejects.
- **Cleanup:** None.

### INV-007 — Ship transfer with insufficient stock
- **Area:** Inventory
- **Title:** Shipping more than available is blocked
- **Priority:** P1
- **Test Type:** Negative, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A transfer requests more than available at source.
- **Route:** `/admin/transfers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/transfers`.
  3. Ship a transfer with insufficient stock.
- **Expected UI Result:** Error `Insufficient stock to ship item`.
- **Expected Backend / Data Result:** `transitionTransfer` validates availability.
- **Negative Assertions:** No overship.
- **Cleanup:** None.

### INV-008 — Stock count with variance
- **Area:** Inventory
- **Title:** Stock count posts a variance adjustment
- **Priority:** P1
- **Test Type:** Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A location with stock.
- **Route:** `/admin/stock-counts`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/stock-counts`.
  3. Create a count.
  4. Enter counted quantities.
  5. Approve the count.
- **Expected UI Result:** Count posts; variance adjusts balances.
- **Expected Backend / Data Result:** `stock_counts`/`stock_count_items`; balances updated to counted values; `inventory_movements` (count).
- **Negative Assertions:** Balances match counted values.
- **Cleanup:** None.

### INV-009 — Low stock alert
- **Area:** Inventory
- **Title:** Low stock triggers an alert
- **Priority:** P1
- **Test Type:** Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A variant with stock at/below its threshold.
- **Route:** `/admin/inventory`, `/admin`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/inventory`.
  3. Verify low-stock items are flagged.
- **Expected UI Result:** Low-stock badge/alert shown.
- **Expected Backend / Data Result:** `ai_alerts` row (low_stock) created.
- **Cleanup:** None.

### INV-010 — Inventory valuation report
- **Area:** Inventory
- **Title:** Valuation report computes inventory value
- **Priority:** P1
- **Test Type:** Integration
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/inventory-reports`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/inventory-reports`.
  3. Open the Valuation tab.
- **Expected UI Result:** Valuation shown (on-hand × avg cost).
- **Expected Backend / Data Result:** `valuation` aggregates balances × costs.
- **Cleanup:** None.

---

## 13. Purchasing Test Cases

### PURCHASE-001 — Create a supplier
- **Area:** Purchasing
- **Title:** Owner creates a supplier
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Test Data:** `QA Supplier One`, email `qa.supplier1@example.com`.
- **Route:** `/admin/suppliers`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/suppliers`.
  3. Create a supplier.
- **Expected UI Result:** Supplier appears.
- **Expected Backend / Data Result:** `suppliers` row created.
- **Cleanup:** Delete the QA supplier.

### PURCHASE-002 — Create a multi-line purchase order
- **Area:** Purchasing
- **Title:** Create a PO with multiple products
- **Priority:** P0
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A supplier and products exist.
- **Test Data:** PO with 2 lines: product A qty 10 @ 50, product B qty 5 @ 20.
- **Route:** `/admin/purchasing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing`.
  3. Create a PO with 2 lines.
- **Expected UI Result:** PO created with running total.
- **Expected Backend / Data Result:** `purchase_orders` row (status `draft`, total 600); `purchase_order_items` rows.
- **Cleanup:** Cancel the PO.

### PURCHASE-003 — Edit a PO before receiving
- **Area:** Purchasing
- **Title:** Edit a draft PO's lines
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A draft PO exists.
- **Route:** `/admin/purchasing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing`.
  3. Edit the draft PO's lines.
- **Expected UI Result:** PO updated.
- **Expected Backend / Data Result:** `purchase_order_items` replaced.
- **Cleanup:** None.

### PURCHASE-004 — Cancel a PO before receiving
- **Area:** Purchasing
- **Title:** Cancel a draft PO
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A draft PO exists.
- **Route:** `/admin/purchasing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing`.
  3. Cancel the PO.
- **Expected UI Result:** PO status `cancelled`.
- **Expected Backend / Data Result:** `purchase_orders.status` → `cancelled`.
- **Cleanup:** None.

### PURCHASE-005 — Full goods receipt updates stock and average cost
- **Area:** Purchasing
- **Title:** Receive a PO and update stock/cost
- **Priority:** P0
- **Test Type:** Inventory Integrity, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A PO exists; a location exists.
- **Test Data:** PO line: product A qty 10 @ 50.
- **Route:** `/admin/purchasing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing`.
  3. Receive the PO (full quantity).
- **Expected UI Result:** Goods receipt created; stock increases.
- **Expected Backend / Data Result:**
  - `goods_receipts` row
  - `goods_receipt_items` rows
  - Inventory balance +10
  - `inventory_movements` (purchase, +10)
  - Average cost updated
  - PO status → `received`
- **Negative Assertions:** Inventory received exactly once.
- **Cleanup:** None.

### PURCHASE-006 — Partial goods receipt
- **Area:** Purchasing
- **Title:** Receive part of a PO
- **Priority:** P1
- **Test Type:** Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A PO for 10 units exists.
- **Route:** `/admin/purchasing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing`.
  3. Receive 4 of 10 units.
- **Expected UI Result:** Partial receipt; PO status `partial`.
- **Expected Backend / Data Result:** Balance +4; `received_quantity` 4.
- **Negative Assertions:** Only 4 units received.
- **Cleanup:** None.

### PURCHASE-007 — Over-receive prevention
- **Area:** Purchasing
- **Title:** Receiving more than ordered is blocked
- **Priority:** P1
- **Test Type:** Negative, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A PO for 10 units exists.
- **Route:** `/admin/purchasing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing`.
  3. Attempt to receive 15 units.
- **Expected UI Result:** Over-receive blocked (unless `allow_over_receipt` setting).
- **Expected Backend / Data Result:** Receiving validates against ordered quantity.
- **Negative Assertions:** No over-receipt.
- **Cleanup:** None.

### PURCHASE-008 — Supplier invoice and payment
- **Area:** Purchasing
- **Title:** Post a supplier invoice and record a payment
- **Priority:** P0
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A supplier exists.
- **Test Data:** Invoice total 600.
- **Route:** `/admin/purchasing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing`.
  3. Create and post a supplier invoice.
  4. Record a payment.
- **Expected UI Result:** Invoice posted; payment recorded.
- **Expected Backend / Data Result:**
  - `supplier_invoices` row (status `posted`)
  - Journal: Dr Inventory/Purchases 600, Cr AP 600
  - `supplier_payments` row
  - Journal: Dr AP, Cr Cash/Bank
  - Invoice status → `paid`
- **Negative Assertions:** Payment does not exceed outstanding; journal balanced.
- **Cleanup:** None.

### PURCHASE-009 — Supplier payment exceeds outstanding
- **Area:** Purchasing
- **Title:** Over-payment on a supplier invoice is rejected
- **Priority:** P1
- **Test Type:** Negative, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A posted invoice with outstanding 600.
- **Route:** `/admin/purchasing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing`.
  3. Attempt to pay 700.
- **Expected UI Result:** Error `Payment amount exceeds outstanding balance`.
- **Expected Backend / Data Result:** `recordSupplierPayment` rejects.
- **Negative Assertions:** No over-payment.
- **Cleanup:** None.

### PURCHASE-010 — Duplicate goods receipt prevention
- **Area:** Purchasing
- **Title:** Receiving the same PO twice does not double stock
- **Priority:** P0
- **Test Type:** Concurrency, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A fully-received PO exists.
- **Route:** `/admin/purchasing`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing`.
  3. Attempt to receive the already-received PO again.
- **Expected UI Result:** Receiving blocked (PO status `received`).
- **Expected Backend / Data Result:** No additional stock movement.
- **Negative Assertions:** No double stock receipt.
- **Cleanup:** None.

---

## 14. Sales / Orders Test Cases

### SALES-001 — Sales list with filters
- **Area:** Sales
- **Title:** Sales list filters by channel/payment/date
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/sales`
- **Selectors / UI Targets:** textbox `Search order # / email / phone`, selects `All channels`, `All payments`, date inputs.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sales`.
  3. Apply filters.
- **Expected UI Result:** Sales list filters correctly.
- **Expected Backend / Data Result:** Server-side paginated sales query.
- **Cleanup:** None.

### SALES-002 — Sale detail view
- **Area:** Sales
- **Title:** View a sale's full detail
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A sale exists.
- **Route:** `/admin/sales`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sales`.
  3. Open a sale's detail.
- **Expected UI Result:** Line items, tenders, totals, customer, cashier, returns shown.
- **Expected Backend / Data Result:** `getSaleDetail` returns full detail.
- **Cleanup:** None.

### SALES-003 — Record a payment on a credit sale
- **Area:** Sales
- **Title:** Settle a credit sale with a payment
- **Priority:** P0
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** An unpaid/partial credit sale exists.
- **Route:** `/admin/sales`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sales`.
  3. Open the credit sale.
  4. Record a payment.
- **Expected UI Result:** Payment recorded; order becomes paid.
- **Expected Backend / Data Result:**
  - `payments` row
  - `orders.paid_total` updated; `payment_status` → `paid`
  - Journal: Dr Cash/Bank, Cr AR
- **Negative Assertions:** Payment does not exceed outstanding; journal balanced.
- **Cleanup:** None.

### SALES-004 — Duplicate payment protection
- **Area:** Sales
- **Title:** Recording a payment on a fully-paid sale is rejected
- **Priority:** P0
- **Test Type:** Negative, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A fully-paid sale exists.
- **Route:** `/admin/sales`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sales`.
  3. Open the paid sale.
  4. Attempt to record another payment.
- **Expected UI Result:** Error `This sale is already fully paid.`
- **Expected Backend / Data Result:** `recordSalePayment` rejects.
- **Negative Assertions:** No duplicate payment.
- **Cleanup:** None.

### SALES-005 — Set fulfillment status
- **Area:** Sales
- **Title:** Update an order's delivery status
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** An order exists.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Update the fulfillment status.
- **Expected UI Result:** Status updates.
- **Expected Backend / Data Result:** `setOrderFulfillment` updates `fulfillment_status`.
- **Negative Assertions:** No inventory/accounting effect.
- **Cleanup:** None.

---

## 15. Accounting / Finance Test Cases

### ACCOUNT-001 — Chart of accounts loads
- **Area:** Accounting
- **Title:** Chart of accounts renders
- **Priority:** P1
- **Test Type:** Smoke
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/accounts`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounts`.
- **Expected UI Result:** Accounts listed.
- **Expected Backend / Data Result:** `accounts` rows.
- **Cleanup:** None.

### ACCOUNT-002 — Trial balance is balanced
- **Area:** Accounting
- **Title:** Trial balance debits equal credits
- **Priority:** P0
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** Sales/purchases/expenses exist.
- **Route:** `/admin/accounting` (Trial Balance tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Open Trial Balance.
- **Expected UI Result:** Total debits = total credits; `balanced` true.
- **Expected Backend / Data Result:** `trialBalance` computes totals.
- **Negative Assertions:** Debits must equal credits.
- **Cleanup:** None.

### ACCOUNT-003 — P&L report
- **Area:** Accounting
- **Title:** P&L shows income, expenses, net profit
- **Priority:** P1
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/accounting` (P&L tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Open P&L.
- **Expected UI Result:** Income, expense, net profit shown.
- **Expected Backend / Data Result:** `profitAndLoss` computes from journal lines.
- **Cleanup:** None.

### ACCOUNT-004 — Balance sheet balances
- **Area:** Accounting
- **Title:** Balance sheet assets = liabilities + equity
- **Priority:** P0
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/accounting` (Balance Sheet tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Open Balance Sheet.
- **Expected UI Result:** `balanced` true.
- **Expected Backend / Data Result:** `balanceSheet` computes; `balanced` flag.
- **Negative Assertions:** Assets must equal liabilities + equity.
- **Cleanup:** None.

### ACCOUNT-005 — Cash flow statement
- **Area:** Accounting
- **Title:** Cash flow shows movements
- **Priority:** P1
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/accounting` (Cash Flow tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Open Cash Flow.
- **Expected UI Result:** Cash movements shown.
- **Expected Backend / Data Result:** `cashFlow` computes.
- **Cleanup:** None.

### ACCOUNT-006 — General ledger
- **Area:** Accounting
- **Title:** General ledger lists entries with running balance
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/accounting` (General Ledger tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Open General Ledger.
- **Expected UI Result:** Ledger rows with running balance.
- **Expected Backend / Data Result:** `generalLedger` computes.
- **Cleanup:** None.

### ACCOUNT-007 — AR ageing
- **Area:** Accounting
- **Title:** AR ageing buckets outstanding receivables
- **Priority:** P1
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** Unpaid/partial orders exist.
- **Route:** `/admin/accounting` (AR Ageing tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Open AR Ageing.
- **Expected UI Result:** Outstanding receivables bucketed by age.
- **Expected Backend / Data Result:** `arAgeing` computes.
- **Cleanup:** None.

### ACCOUNT-008 — AP ageing
- **Area:** Accounting
- **Title:** AP ageing buckets outstanding payables
- **Priority:** P1
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** Posted supplier invoices exist.
- **Route:** `/admin/accounting` (AP Ageing tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Open AP Ageing.
- **Expected UI Result:** Outstanding payables bucketed by age.
- **Expected Backend / Data Result:** `apAgeing` computes.
- **Cleanup:** None.

### ACCOUNT-009 — Balance validation
- **Area:** Accounting
- **Title:** Balance validation reports no unbalanced entries
- **Priority:** P0
- **Test Type:** Financial Integrity, Data Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/accounting` (Overview)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. View the balance validation.
- **Expected UI Result:** `balanced` true; no unbalanced entries.
- **Expected Backend / Data Result:** `balanceValidation` checks every entry.
- **Negative Assertions:** No unbalanced journal entries.
- **Cleanup:** None.

### ACCOUNT-010 — Record an expense
- **Area:** Accounting
- **Title:** Record an expense and post the journal
- **Priority:** P0
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** An expense category exists.
- **Test Data:** Expense `100`, category `Rent`, cash.
- **Route:** `/admin/accounting`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Record an expense.
- **Expected UI Result:** Expense recorded.
- **Expected Backend / Data Result:**
  - `expenses` row (status `approved`)
  - Journal: Dr Expense account 100, Cr Cash 100
- **Negative Assertions:** Journal balanced.
- **Cleanup:** None.

### ACCOUNT-011 — Expense approval flow
- **Area:** Accounting
- **Title:** Expense requiring approval is held until approved
- **Priority:** P1
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** Shop setting `expenses.require_approval` = true.
- **Route:** `/admin/accounting`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Record an expense (held as pending).
  4. Approve it.
- **Expected UI Result:** Expense posts after approval.
- **Expected Backend / Data Result:** `expenses.status` pending → approved; journal posted on approval.
- **Negative Assertions:** No journal before approval.
- **Cleanup:** None.

### ACCOUNT-012 — Duplicate expense submission
- **Area:** Accounting
- **Title:** Double-clicking record expense creates one expense
- **Priority:** P1
- **Test Type:** Concurrency
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/accounting`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Click record expense twice.
- **Expected UI Result:** One expense created; button disabled while busy.
- **Expected Backend / Data Result:** Exactly one `expenses` row.
- **Negative Assertions:** No duplicate expense.
- **Cleanup:** None.

---

## 16. Expenses Test Cases

### EXPENSE-001 — Expense validation (amount > 0)
- **Area:** Expenses
- **Title:** Zero/negative expense amount rejected
- **Priority:** P1
- **Test Type:** Validation, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/accounting`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Attempt to record an expense with amount 0 or negative.
- **Expected UI Result:** Error `Amount must be greater than zero`.
- **Expected Backend / Data Result:** `recordExpenseAndPost` rejects.
- **Cleanup:** None.

### EXPENSE-002 — Expense attachment
- **Area:** Expenses
- **Title:** Attach a file to an expense
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/accounting`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/accounting`.
  3. Record an expense with an attachment.
- **Expected UI Result:** Attachment saved.
- **Expected Backend / Data Result:** `expenses.attachment_url` set.
- **Cleanup:** None.

---

## 17. Storefront Admin Test Cases

### STORE-ADMIN-001 — Edit storefront theme
- **Area:** Storefront Admin
- **Title:** Owner edits theme colours
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/storefront` (Theme tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront`.
  3. Edit a theme colour.
  4. Save.
- **Expected UI Result:** Draft saved.
- **Expected Backend / Data Result:** `storefront_settings.draft` updated.
- **Cleanup:** Discard the draft.

### STORE-ADMIN-002 — Publish storefront
- **Area:** Storefront Admin
- **Title:** Publish promotes draft to live
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/storefront`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront`.
  3. Make a draft change.
  4. Publish.
- **Expected UI Result:** Draft promoted to live.
- **Expected Backend / Data Result:** `storefront_settings` live columns updated; `is_draft` cleared on content rows.
- **Cleanup:** None.

### STORE-ADMIN-003 — Discard storefront draft
- **Area:** Storefront Admin
- **Title:** Discard reverts draft changes
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/storefront`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront`.
  3. Make a draft change.
  4. Discard.
- **Expected UI Result:** Draft cleared.
- **Expected Backend / Data Result:** `storefront_settings.draft` cleared; draft-only content rows deleted.
- **Cleanup:** None.

### STORE-ADMIN-004 — Manage banners
- **Area:** Storefront Admin
- **Title:** Create/edit/delete a banner
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/storefront` (Banners tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront`.
  3. Create a banner.
  4. Edit it.
  5. Delete it.
- **Expected UI Result:** Banner CRUD works.
- **Expected Backend / Data Result:** `storefront_banners` rows.
- **Cleanup:** None.

### STORE-ADMIN-005 — Manage navigation
- **Area:** Storefront Admin
- **Title:** Create/edit/delete a nav item
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/storefront` (Navigation tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront`.
  3. Create a nav item.
  4. Edit it.
  5. Delete it.
- **Expected UI Result:** Nav CRUD works.
- **Expected Backend / Data Result:** `storefront_navigation` rows.
- **Cleanup:** None.

### STORE-ADMIN-006 — Manage policies
- **Area:** Storefront Admin
- **Title:** Edit a store policy
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/storefront` (Policies tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront`.
  3. Edit a policy.
- **Expected UI Result:** Policy updated.
- **Expected Backend / Data Result:** `storefront_policies` upserted.
- **Cleanup:** None.

### STORE-ADMIN-007 — Manage CMS pages
- **Area:** Storefront Admin
- **Title:** Create/edit/delete a CMS page
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/storefront` (Pages tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront`.
  3. Create a page.
  4. Edit it.
  5. Delete it.
- **Expected UI Result:** Page CRUD works.
- **Expected Backend / Data Result:** `storefront_pages` rows.
- **Cleanup:** None.

---

## 18. Public Storefront Test Cases

### STORE-001 — Storefront home loads
- **Area:** Storefront
- **Title:** Home page renders with hero and sections
- **Priority:** P1
- **Test Type:** Smoke
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>`
- **Steps:**
  1. As a guest, navigate to `/store/<slug>`.
- **Expected UI Result:** Header, hero, product sections, footer render.
- **Expected Backend / Data Result:** `getStorefront` returns shop data.
- **Cleanup:** None.

### STORE-002 — Shop page with filters
- **Area:** Storefront
- **Title:** Shop page filters by category/brand/search
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/shop`
- **Steps:**
  1. As a guest, navigate to `/store/<slug>/shop`.
  2. Apply a category filter.
  3. Search.
- **Expected UI Result:** Products filter correctly.
- **Expected Backend / Data Result:** `listStorefrontProducts` filters.
- **Cleanup:** None.

### STORE-003 — Product detail page
- **Area:** Storefront
- **Title:** Product detail shows price, stock, add-to-cart
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/product/<slug>`
- **Steps:**
  1. As a guest, navigate to a product page.
- **Expected UI Result:** Price, stock, quantity, Add to Cart, Wishlist, Compare shown.
- **Expected Backend / Data Result:** `getStorefrontProduct` returns product + stock.
- **Cleanup:** None.

### STORE-004 — Product variant selection
- **Area:** Storefront
- **Title:** Selecting a variant updates price/stock
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Preconditions:** A multi-variant product exists.
- **Route:** `/store/<slug>/product/<slug>`
- **Steps:**
  1. As a guest, navigate to a multi-variant product.
  2. Select a different variant.
- **Expected UI Result:** Price/stock update.
- **Expected Backend / Data Result:** None (client-side).
- **Cleanup:** None.

### STORE-005 — Add to cart
- **Area:** Storefront
- **Title:** Add a product to the cart
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/product/<slug>`
- **Steps:**
  1. As a guest, navigate to a product.
  2. Click `Add to Cart`.
- **Expected UI Result:** Cart count increments.
- **Expected Backend / Data Result:** Cart stored in localStorage.
- **Cleanup:** Clear the cart.

### STORE-006 — Cart quantity and remove
- **Area:** Storefront
- **Title:** Cart quantity steppers and removal
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/cart`
- **Steps:**
  1. As a guest, add products to cart.
  2. Navigate to `/store/<slug>/cart`.
  3. Change quantity.
  4. Remove an item.
- **Expected UI Result:** Cart updates.
- **Expected Backend / Data Result:** None.
- **Cleanup:** Clear the cart.

### STORE-007 — Wishlist
- **Area:** Storefront
- **Title:** Add/remove from wishlist
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/wishlist`
- **Steps:**
  1. As a guest, add a product to the wishlist.
  2. Navigate to `/store/<slug>/wishlist`.
  3. Remove it.
- **Expected UI Result:** Wishlist updates.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### STORE-008 — Compare
- **Area:** Storefront
- **Title:** Add/remove from compare
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/compare`
- **Steps:**
  1. As a guest, add products to compare.
  2. Navigate to `/store/<slug>/compare`.
  3. Remove one.
- **Expected UI Result:** Compare updates.
- **Cleanup:** None.

### STORE-009 — Invalid product slug
- **Area:** Storefront
- **Title:** Unknown product slug shows not-found
- **Priority:** P2
- **Test Type:** Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/product/does-not-exist`
- **Steps:**
  1. As a guest, navigate to an invalid product slug.
- **Expected UI Result:** `Product not found` empty state.
- **Expected Backend / Data Result:** `getStorefrontProduct` returns null.
- **Cleanup:** None.

### STORE-010 — Invalid shop slug
- **Area:** Storefront
- **Title:** Unknown shop slug shows store unavailable
- **Priority:** P2
- **Test Type:** Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/does-not-exist`
- **Steps:**
  1. As a guest, navigate to an invalid shop slug.
- **Expected UI Result:** `Store unavailable` / `Store does not exist`.
- **Expected Backend / Data Result:** `getStorefront` returns null.
- **Cleanup:** None.

### STORE-011 — Cost price never exposed publicly
- **Area:** Storefront
- **Title:** Public pages never expose cost price
- **Priority:** P0
- **Test Type:** Security
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/product/<slug>`, `/store/<slug>/shop`
- **Steps:**
  1. As a guest, navigate to product/shop pages.
  2. Inspect HTML and network responses for `cost`, `unit_cost`, `cost_price`, `avg_cost`.
- **Expected UI Result:** No cost data present.
- **Expected Backend / Data Result:** Storefront queries never select `variant_costs`.
- **Negative Assertions:** Cost must never leak.
- **Cleanup:** None.

### STORE-012 — Storefront language switcher (RTL)
- **Area:** Storefront
- **Title:** Switching to Arabic/Urdu enables RTL
- **Priority:** P1
- **Test Type:** Responsive, Integration
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>`
- **Selectors / UI Targets:** combobox `Language`.
- **Steps:**
  1. As a guest, navigate to `/store/<slug>`.
  2. Switch language to Arabic.
- **Expected UI Result:** `document.dir` becomes `rtl`; labels translate.
- **Expected Backend / Data Result:** None.
- **Cleanup:** Switch back to English.

### STORE-013 — Storefront responsive (no horizontal overflow)
- **Area:** Storefront
- **Title:** No horizontal overflow at mobile widths
- **Priority:** P1
- **Test Type:** Responsive
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** All storefront pages
- **Steps:**
  1. As a guest, navigate to each storefront page at 375px.
- **Expected UI Result:** No horizontal overflow.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.
- **Notes for Playwright:** Already covered by `storefront-responsive.spec.ts`; reference it.

---

## 19. Ecommerce Checkout Test Cases

### CHECKOUT-001 — COD checkout creates an unpaid order
- **Area:** Checkout
- **Title:** Guest COD order is created unpaid
- **Priority:** P0
- **Test Type:** Happy Path, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** A product in stock; guest checkout enabled.
- **Route:** `/store/<slug>/checkout`
- **Steps:**
  1. As a guest, add a product to cart.
  2. Navigate to `/store/<slug>/checkout`.
  3. Fill billing details.
  4. Select `Cash on delivery`.
  5. Place the order.
- **Expected UI Result:** Order placed; success page shows order number.
- **Expected Backend / Data Result:**
  - `orders` row (channel `online`, status `confirmed`, payment_status `unpaid`, fulfillment_status `unfulfilled`)
  - Inventory deducted
  - Journal: Dr AR (1150), Cr Revenue + Tax
  - `ai_alerts` row (new_order)
- **Negative Assertions:** COD order is NOT marked paid.
- **Cleanup:** None.

### CHECKOUT-002 — Bank transfer checkout shows bank details
- **Area:** Checkout
- **Title:** Bank transfer method shows merchant bank details
- **Priority:** P0
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Preconditions:** Shop has bank transfer settings configured.
- **Route:** `/store/<slug>/checkout`
- **Steps:**
  1. As a guest, add a product to cart.
  2. Navigate to `/store/<slug>/checkout`.
  3. Select `Bank transfer`.
- **Expected UI Result:** Bank details (bank name, account title, number, IBAN) shown.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### CHECKOUT-003 — Bank transfer order is created unpaid
- **Area:** Checkout
- **Title:** Bank transfer order is unpaid until proof verified
- **Priority:** P0
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Route:** `/store/<slug>/checkout`
- **Steps:**
  1. As a guest, add a product to cart.
  2. Navigate to `/store/<slug>/checkout`.
  3. Select `Bank transfer`.
  4. Place the order.
- **Expected UI Result:** Order placed; success page shows proof upload.
- **Expected Backend / Data Result:** `orders` row (payment_status `unpaid`); journal Dr AR.
- **Negative Assertions:** Order is NOT paid.
- **Cleanup:** None.

### CHECKOUT-004 — Checkout validation (name/email required)
- **Area:** Checkout
- **Title:** Missing name/email blocks checkout
- **Priority:** P1
- **Test Type:** Validation, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/checkout`
- **Steps:**
  1. As a guest, add a product to cart.
  2. Navigate to `/store/<slug>/checkout`.
  3. Leave name/email empty.
  4. Attempt to place the order.
- **Expected UI Result:** Error `Name and email are required`.
- **Expected Backend / Data Result:** No order.
- **Cleanup:** None.

### CHECKOUT-005 — Double-click Place Order creates one order
- **Area:** Checkout
- **Title:** Duplicate checkout submission prevented
- **Priority:** P0
- **Test Type:** Concurrency
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Route:** `/store/<slug>/checkout`
- **Steps:**
  1. As a guest, add a product to cart.
  2. Navigate to `/store/<slug>/checkout`.
  3. Fill details.
  4. Click `Place Order` twice.
- **Expected UI Result:** One order created; button shows `Placing order…`.
- **Expected Backend / Data Result:** Exactly one `orders` row (deduped by `client_uuid`).
- **Negative Assertions:** No duplicate order; no double inventory deduction.
- **Cleanup:** None.

### CHECKOUT-006 — Empty cart checkout redirects
- **Area:** Checkout
- **Title:** Checkout with empty cart shows empty state
- **Priority:** P2
- **Test Type:** Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/checkout`
- **Steps:**
  1. As a guest, navigate to `/store/<slug>/checkout` with an empty cart.
- **Expected UI Result:** Empty cart message shown.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### CHECKOUT-007 — Track order
- **Area:** Checkout
- **Title:** Track an order by number + contact
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** An order exists with a contact email.
- **Route:** `/store/<slug>/track-order`
- **Steps:**
  1. As a guest, navigate to `/store/<slug>/track-order`.
  2. Enter the order number and contact email.
  3. Submit.
- **Expected UI Result:** Order status shown.
- **Expected Backend / Data Result:** `trackOrder` returns the order.
- **Negative Assertions:** Wrong contact does not reveal the order.
- **Cleanup:** None.

### CHECKOUT-008 — Track order with wrong contact
- **Area:** Checkout
- **Title:** Wrong contact does not reveal an order
- **Priority:** P1
- **Test Type:** Negative, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** An order exists.
- **Route:** `/store/<slug>/track-order`
- **Steps:**
  1. As a guest, navigate to `/store/<slug>/track-order`.
  2. Enter a valid order number but a wrong contact.
- **Expected UI Result:** `No matching order found`.
- **Expected Backend / Data Result:** `trackOrder` returns `found: false`.
- **Negative Assertions:** Order not revealed to wrong contact.
- **Cleanup:** None.

---

## 20. Payment Proof Test Cases (P0)

> **Critical:** A screenshot upload must NEVER mark an order paid. Only merchant verification does.

### PROOF-001 — Upload a payment proof
- **Area:** Payment Proof
- **Title:** Customer uploads a bank-transfer screenshot
- **Priority:** P0
- **Test Type:** Happy Path, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** A bank-transfer order exists (unpaid).
- **Route:** `/store/<slug>/payment/success`
- **Steps:**
  1. As a guest, place a bank-transfer order.
  2. On the success page, upload a proof file.
- **Expected UI Result:** `Payment proof submitted. The shop will verify it before dispatch.`
- **Expected Backend / Data Result:**
  - File uploaded to the private `payment-proofs` bucket
  - `payment_proofs` row created (status `submitted`)
- **Negative Assertions:** The order is STILL unpaid after upload.
- **Cleanup:** None.
- **Notes for Playwright:** Verify `orders.payment_status` is still `unpaid` after upload.

### PROOF-002 — Upload does not mark order paid
- **Area:** Payment Proof
- **Title:** Proof submission leaves the order unpaid
- **Priority:** P0
- **Test Type:** Financial Integrity, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** A bank-transfer order exists.
- **Route:** `/store/<slug>/payment/success`
- **Steps:**
  1. As a guest, place a bank-transfer order.
  2. Upload a proof.
- **Expected Backend / Data Result:** `orders.payment_status` remains `unpaid`; no `payments` row; no journal.
- **Negative Assertions:** Upload must NOT create a payment or mark paid.
- **Cleanup:** None.

### PROOF-003 — Merchant verifies a proof (marks paid)
- **Area:** Payment Proof
- **Title:** Merchant verification marks the order paid
- **Priority:** P0
- **Test Type:** Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A submitted proof exists for an unpaid order.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Open the order with the submitted proof.
  4. Click `Verify & mark paid`.
- **Expected UI Result:** Proof status `verified`; order becomes paid.
- **Expected Backend / Data Result:**
  - `payment_proofs.status` → `verified`
  - `orders.payment_status` → `paid`; `paid_total` updated
  - `payments` row created
  - Journal: Dr Bank, Cr AR
- **Negative Assertions:** Payment created exactly once.
- **Cleanup:** None.

### PROOF-004 — Merchant rejects a proof
- **Area:** Payment Proof
- **Title:** Merchant rejection with a reason
- **Priority:** P0
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A submitted proof exists.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Open the order with the submitted proof.
  4. Click `Reject`, enter a reason.
- **Expected UI Result:** Proof status `rejected`; reason shown.
- **Expected Backend / Data Result:** `payment_proofs.status` → `rejected`; `rejection_reason` set.
- **Negative Assertions:** Order remains unpaid.
- **Cleanup:** None.

### PROOF-005 — Customer re-uploads after rejection
- **Area:** Payment Proof
- **Title:** Re-upload after rejection
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** A rejected proof exists.
- **Route:** `/store/<slug>/payment/success`
- **Steps:**
  1. As a guest, navigate to the order's success page.
  2. Upload a new proof.
- **Expected UI Result:** New proof submitted.
- **Expected Backend / Data Result:** A new `payment_proofs` row (status `submitted`).
- **Cleanup:** None.

### PROOF-006 — Duplicate verification protection
- **Area:** Payment Proof
- **Title:** Verifying an already-paid order does not double-pay
- **Priority:** P0
- **Test Type:** Concurrency, Financial Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A verified proof on a paid order.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Attempt to verify again.
- **Expected UI Result:** No additional payment (order already paid).
- **Expected Backend / Data Result:** `reviewPaymentProof` skips payment if already paid.
- **Negative Assertions:** No duplicate payment.
- **Cleanup:** None.

### PROOF-007 — Unauthorized role cannot view proofs
- **Area:** Payment Proof
- **Title:** Cashier cannot preview payment proofs
- **Priority:** P0
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Cashier
- **Preconditions:** A submitted proof exists.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as cashier.
  2. Navigate to `/admin/orders`.
  3. Attempt to preview a proof.
- **Expected UI Result:** Denied.
- **Expected Backend / Data Result:** `getPaymentProofUrl` requires `MANAGER_ROLES`.
- **Negative Assertions:** Cashier must not view proofs.
- **Cleanup:** None.

### PROOF-008 — Another tenant cannot view a proof
- **Area:** Payment Proof
- **Title:** Cross-tenant proof access denied
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Preconditions:** A Shop B proof exists.
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Attempt to fetch a Shop B proof by ID.
- **Expected UI Result:** `Proof not found`.
- **Expected Backend / Data Result:** `getPaymentProofUrl` filters by `shop_id`.
- **Negative Assertions:** Shop B proof must not be accessible.
- **Cleanup:** None.

### PROOF-009 — Invalid file type rejected
- **Area:** Payment Proof
- **Title:** Non-image/PDF upload rejected
- **Priority:** P1
- **Test Type:** Validation, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Preconditions:** A bank-transfer order exists.
- **Route:** `/store/<slug>/payment/success`
- **Steps:**
  1. As a guest, place a bank-transfer order.
  2. Upload a `.txt` file.
- **Expected UI Result:** Error `Upload a JPG, PNG, WEBP or PDF.`
- **Expected Backend / Data Result:** `submitPaymentProof` rejects.
- **Cleanup:** None.

### PROOF-010 — Oversized file rejected
- **Area:** Payment Proof
- **Title:** File over 5MB rejected
- **Priority:** P2
- **Test Type:** Validation, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Preconditions:** A bank-transfer order exists.
- **Route:** `/store/<slug>/payment/success`
- **Steps:**
  1. As a guest, place a bank-transfer order.
  2. Upload a file over 5MB.
- **Expected UI Result:** Error `File too large (max 5MB).`
- **Expected Backend / Data Result:** `submitPaymentProof` rejects.
- **Cleanup:** None.

---

## 21. Custom Domains Test Cases

### DOMAIN-001 — Add a domain
- **Area:** Domains
- **Title:** Owner adds a custom domain
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Test Data:** `qa-${RUN_ID}.example.com`
- **Route:** `/admin/storefront` (Domains tab)
- **Selectors / UI Targets:** field `Add a domain`, button `Add domain`.
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront` → Domains.
  3. Enter a domain.
  4. Click `Add domain`.
- **Expected UI Result:** Domain added as `pending`; DNS instructions shown.
- **Expected Backend / Data Result:** `shop_domains` row (verification_status `pending`); audit log.
- **Cleanup:** Remove the domain.

### DOMAIN-002 — Duplicate domain rejected
- **Area:** Domains
- **Title:** Adding an in-use domain fails
- **Priority:** P1
- **Test Type:** Negative, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A domain is already added.
- **Route:** `/admin/storefront` (Domains tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront` → Domains.
  3. Add the same domain again.
- **Expected UI Result:** Error `This domain is already added.` (same shop) or `This domain is already in use.` (other shop).
- **Expected Backend / Data Result:** `addDomain` rejects.
- **Cleanup:** None.

### DOMAIN-003 — Invalid domain rejected
- **Area:** Domains
- **Title:** Malformed domain rejected
- **Priority:** P2
- **Test Type:** Validation, Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/storefront` (Domains tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront` → Domains.
  3. Add an invalid domain (e.g. `not a domain`).
- **Expected UI Result:** Error `Enter a valid domain, e.g. trendies.pk or shop.trendies.pk`.
- **Expected Backend / Data Result:** `addDomain` rejects.
- **Cleanup:** None.

### DOMAIN-004 — Mark domain verified
- **Area:** Domains
- **Title:** Owner marks a domain verified
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A pending domain exists.
- **Route:** `/admin/storefront` (Domains tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront` → Domains.
  3. Click `Mark verified`.
- **Expected UI Result:** Domain status `verified`.
- **Expected Backend / Data Result:** `shop_domains.verification_status` → `verified`; audit log.
- **Cleanup:** None.

### DOMAIN-005 — Make primary requires verified
- **Area:** Domains
- **Title:** Unverified domain cannot be made primary
- **Priority:** P1
- **Test Type:** Negative, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A pending (unverified) domain exists.
- **Route:** `/admin/storefront` (Domains tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront` → Domains.
  3. Attempt to make the unverified domain primary.
- **Expected UI Result:** Error `Verify the domain before making it primary.`
- **Expected Backend / Data Result:** `setPrimaryDomain` rejects.
- **Cleanup:** None.

### DOMAIN-006 — Make primary
- **Area:** Domains
- **Title:** Verified domain becomes primary
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A verified domain exists.
- **Route:** `/admin/storefront` (Domains tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront` → Domains.
  3. Click `Make primary`.
- **Expected UI Result:** Domain becomes primary.
- **Expected Backend / Data Result:** `shop_domains.is_primary` updated (only one primary).
- **Cleanup:** None.

### DOMAIN-007 — Remove a domain
- **Area:** Domains
- **Title:** Owner removes a domain
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Preconditions:** A domain exists.
- **Route:** `/admin/storefront` (Domains tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/storefront` → Domains.
  3. Remove the domain.
- **Expected UI Result:** Domain removed.
- **Expected Backend / Data Result:** `shop_domains` row deleted; audit log.
- **Cleanup:** None.

### DOMAIN-008 — Unknown hostname does not resolve
- **Area:** Domains
- **Title:** Unknown/unverified host never serves a storefront
- **Priority:** P0
- **Test Type:** Security, Tenant Isolation
- **Suitable for Playwright?:** Partial (host simulation)
- **Roles:** Guest
- **Preconditions:** None.
- **Steps:**
  1. Simulate a request with an unknown Host header.
- **Expected UI Result:** No storefront served (falls back to default routing).
- **Expected Backend / Data Result:** `resolveShopSlugForHost` returns null for unknown/unverified hosts.
- **Negative Assertions:** Unknown host must not resolve to any shop.
- **Cleanup:** None.
- **Notes for Playwright:** Requires host-header simulation; separate from DNS tests.

### DOMAIN-009 — Cross-shop domain isolation
- **Area:** Domains
- **Title:** A domain belongs to exactly one shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A domain is added to Shop A.
- **Route:** `/admin/storefront` (Domains tab)
- **Steps:**
  1. Authenticate as Shop B owner.
  2. Navigate to `/admin/storefront` → Domains.
  3. Attempt to add Shop A's domain.
- **Expected UI Result:** Error `This domain is already in use.`
- **Expected Backend / Data Result:** `addDomain` rejects cross-shop duplicates.
- **Negative Assertions:** A domain cannot serve two shops.
- **Cleanup:** None.

---

## 22. AI Business Assistant Test Cases

### AI-001 — Ask a business-data question
- **Area:** AI
- **Title:** Assistant answers a sales question
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Owner
- **Route:** `/admin/ai`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/ai`.
  3. Ask "How did sales go over the last 7 days?"
- **Expected UI Result:** An answer is returned.
- **Expected Backend / Data Result:** `askAssistant` runs tools.
- **Cleanup:** None.

### AI-002 — AI action approval flow
- **Area:** AI
- **Title:** Approve an AI write action
- **Priority:** P1
- **Test Type:** Integration
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Owner
- **Preconditions:** AI write actions enabled (approval mode).
- **Route:** `/admin/ai`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/ai`.
  3. Request an AI action (e.g. create PO).
  4. Approve it.
- **Expected UI Result:** Action executes; result posted back.
- **Expected Backend / Data Result:** `executeAiAction` runs the executor; audit log.
- **Cleanup:** None.

### AI-003 — AI action respects RBAC
- **Area:** AI
- **Title:** Unauthorized role cannot execute AI write actions
- **Priority:** P0
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Cashier
- **Route:** `/admin/ai`
- **Steps:**
  1. Authenticate as cashier.
  2. Attempt to execute an AI write action.
- **Expected UI Result:** Denied.
- **Expected Backend / Data Result:** `executeAiAction` requires the action's roles.
- **Negative Assertions:** AI must not bypass RBAC.
- **Cleanup:** None.

### AI-004 — AI action cannot bypass business rules
- **Area:** AI
- **Title:** AI adjust-inventory respects stock rules
- **Priority:** P0
- **Test Type:** Security, Integration
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Owner
- **Preconditions:** AI adjust_inventory enabled.
- **Route:** `/admin/ai`
- **Steps:**
  1. Authenticate as owner.
  2. Request an AI adjust-inventory action with invalid data.
- **Expected UI Result:** Action fails validation.
- **Expected Backend / Data Result:** `adjustInventory` validates; cannot bypass rules.
- **Negative Assertions:** AI cannot bypass business rules.
- **Cleanup:** None.

### AI-005 — AI tenant isolation
- **Area:** AI
- **Title:** AI answers are scoped to the shop
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Owner
- **Preconditions:** Shop A and Shop B exist.
- **Route:** `/admin/ai`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Ask about Shop B data.
- **Expected UI Result:** Answer does not include Shop B data.
- **Expected Backend / Data Result:** AI tools filter by `shop_id`.
- **Negative Assertions:** No cross-tenant AI leakage.
- **Cleanup:** None.

### AI-006 — Knowledge base ingestion and retrieval
- **Area:** AI
- **Title:** Upload a knowledge doc and query it
- **Priority:** P1
- **Test Type:** Integration
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Owner
- **Route:** `/admin/ai` (Knowledge tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/ai` → Knowledge.
  3. Upload a document.
  4. Ask about it.
- **Expected UI Result:** Document ingested; answer cites it.
- **Expected Backend / Data Result:** `ai_knowledge_documents`/`ai_knowledge_chunks` rows.
- **Cleanup:** Delete the document.

### AI-007 — Business memory approval
- **Area:** AI
- **Title:** Business memory fact requires approval
- **Priority:** P2
- **Test Type:** Integration
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Owner
- **Route:** `/admin/ai` (Memory tab)
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/ai` → Memory.
  3. Create a memory fact.
  4. Approve it.
- **Expected UI Result:** Memory approved and active.
- **Expected Backend / Data Result:** `ai_business_memory` status updated.
- **Cleanup:** None.

---

## 23. Storefront AI Salesman Test Cases

### AI-SALES-001 — Guest asks the AI salesman
- **Area:** AI Salesman
- **Title:** Guest can ask the salesman about products
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Guest
- **Route:** `/store/<slug>`
- **Selectors / UI Targets:** AI launcher button, message input.
- **Steps:**
  1. As a guest, navigate to `/store/<slug>`.
  2. Open the AI salesman.
  3. Ask "what do you have under 50?"
- **Expected UI Result:** An answer with products is returned.
- **Expected Backend / Data Result:** `askSalesman` runs tools.
- **Cleanup:** None.

### AI-SALES-002 — Salesman price filter
- **Area:** AI Salesman
- **Title:** Salesman filters by price
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Guest
- **Route:** `/store/<slug>`
- **Steps:**
  1. As a guest, open the AI salesman.
  2. Ask for products under a price.
- **Expected UI Result:** Products within the price range returned.
- **Expected Backend / Data Result:** `search_products` tool filters by price.
- **Cleanup:** None.

### AI-SALES-003 — Salesman cart proposal
- **Area:** AI Salesman
- **Title:** Salesman proposes cart items
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Guest
- **Route:** `/store/<slug>`
- **Steps:**
  1. As a guest, open the AI salesman.
  2. Ask for a recommendation.
  3. Accept the cart proposal.
- **Expected UI Result:** Items added to cart.
- **Expected Backend / Data Result:** `propose_cart` returns variants.
- **Cleanup:** Clear the cart.

### AI-SALES-004 — Salesman excludes internal products
- **Area:** AI Salesman
- **Title:** Internal-only products not returned
- **Priority:** P0
- **Test Type:** Security, Integration
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Guest
- **Preconditions:** An internal (`is_online = false`) product exists.
- **Route:** `/store/<slug>`
- **Steps:**
  1. As a guest, open the AI salesman.
  2. Ask about the internal product.
- **Expected UI Result:** Internal product not returned.
- **Expected Backend / Data Result:** Salesman tools exclude `is_online = false`.
- **Negative Assertions:** Internal product must not be exposed.
- **Cleanup:** None.

### AI-SALES-005 — Salesman does not expose cost
- **Area:** AI Salesman
- **Title:** Cost price not returned by salesman
- **Priority:** P0
- **Test Type:** Security
- **Suitable for Playwright?:** Partial (AI external)
- **Roles:** Guest
- **Route:** `/store/<slug>`
- **Steps:**
  1. As a guest, open the AI salesman.
  2. Ask about cost/margin.
- **Expected UI Result:** Cost not returned.
- **Expected Backend / Data Result:** Salesman tools never return `variant_costs`.
- **Negative Assertions:** Cost must not leak.
- **Cleanup:** None.

---

## 24. Store Data Manager Test Cases (P0)

### DM-001 — Owner can access the Data Manager
- **Area:** Data Manager
- **Title:** Owner opens the Data Manager
- **Priority:** P0
- **Test Type:** Permission
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/data-manager`.
- **Expected UI Result:** Data Manager loads with table list.
- **Expected Backend / Data Result:** `dmTablesFn` returns allowlisted tables.
- **Cleanup:** None.

### DM-002 — Preview a table
- **Area:** Data Manager
- **Title:** Preview rows with conditions
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/data-manager`.
  3. Select a table.
  4. Add a condition.
  5. Preview.
- **Expected UI Result:** Matching rows shown with count.
- **Expected Backend / Data Result:** `dmPreview` returns rows + count.
- **Cleanup:** None.

### DM-003 — Bulk update
- **Area:** Data Manager
- **Title:** Bulk update rows
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** QA rows exist.
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/data-manager`.
  3. Select a table with QA rows.
  4. Preview.
  5. Bulk update a column.
- **Expected UI Result:** Rows updated.
- **Expected Backend / Data Result:** `dmUpdate` updates matching rows; audit log.
- **Cleanup:** Restore the rows.

### DM-004 — Bulk delete requires typed confirmation
- **Area:** Data Manager
- **Title:** Bulk delete requires typing DELETE
- **Priority:** P0
- **Test Type:** Security, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/data-manager`.
  3. Select a table, preview.
  4. Attempt to delete without typing DELETE.
- **Expected UI Result:** Error `Type DELETE to confirm the bulk delete.`
- **Expected Backend / Data Result:** No deletion.
- **Cleanup:** None.

### DM-005 — Bulk delete with confirmation
- **Area:** Data Manager
- **Title:** Bulk delete QA rows
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** QA rows exist.
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/data-manager`.
  3. Select a table with QA rows.
  4. Preview.
  5. Type DELETE and confirm.
- **Expected UI Result:** Rows deleted.
- **Expected Backend / Data Result:** `dmDelete` deletes matching rows; audit log.
- **Cleanup:** None.

### DM-006 — Insert stamps shop_id
- **Area:** Data Manager
- **Title:** Insert always stamps the caller's shop_id
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/data-manager`.
  3. Insert a row (JSON/CSV).
- **Expected UI Result:** Row inserted with the caller's shop_id.
- **Expected Backend / Data Result:** `dmInsert` stamps `shop_id`.
- **Negative Assertions:** Client-supplied shop_id ignored.
- **Cleanup:** Delete the inserted row.

### DM-007 — Protected tables are read-only
- **Area:** Data Manager
- **Title:** shop_users and audit_logs cannot be modified
- **Priority:** P0
- **Test Type:** Security
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/data-manager`.
  3. Attempt to update/delete `shop_users` or `audit_logs`.
- **Expected UI Result:** Error `Table "X" is protected (read-only)`.
- **Expected Backend / Data Result:** `assertTable` rejects writes to protected tables.
- **Negative Assertions:** No modification of protected tables.
- **Cleanup:** None.

### DM-008 — Reset requires owner and typed RESET
- **Area:** Data Manager
- **Title:** Reset requires owner role and typed RESET
- **Priority:** P0
- **Test Type:** Security, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/data-manager`.
  3. Attempt a reset without typing RESET.
- **Expected UI Result:** Error `Type RESET to confirm.`
- **Expected Backend / Data Result:** `dmResetFn` rejects.
- **Cleanup:** None.
- **Notes for Playwright:** Do NOT actually run a destructive reset on shared data. Use an isolated QA shop.

### DM-009 — No raw SQL exposure
- **Area:** Data Manager
- **Title:** Data Manager uses parameter-bound conditions
- **Priority:** P0
- **Test Type:** Security
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/data-manager`.
  3. Attempt to inject SQL into a condition value.
- **Expected UI Result:** No SQL injection; values are parameter-bound.
- **Expected Backend / Data Result:** `applyConditions` validates columns and binds values.
- **Negative Assertions:** No raw SQL execution.
- **Cleanup:** None.

### DM-010 — Cross-tenant attack attempt
- **Area:** Data Manager
- **Title:** Data Manager cannot reach another shop's data
- **Priority:** P0
- **Test Type:** Tenant Isolation, Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Route:** `/admin/data-manager`
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Navigate to `/admin/data-manager`.
  3. Attempt to preview/update Shop B data.
- **Expected UI Result:** Only Shop A rows affected.
- **Expected Backend / Data Result:** Every op forced to `.eq("shop_id", Shop A)`.
- **Negative Assertions:** No cross-tenant access.
- **Cleanup:** None.

---

## 25. Reports Test Cases

### REPORT-001 — Sales reports load
- **Area:** Reports
- **Title:** Sales reports render with totals
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/sales-reports`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sales-reports`.
  3. Open the Summary tab.
- **Expected UI Result:** Sales, orders, AOV, discounts, tax, paid, outstanding shown.
- **Expected Backend / Data Result:** `salesOverview` computes.
- **Cleanup:** None.

### REPORT-002 — Sales by product report
- **Area:** Reports
- **Title:** Sales by product with margin
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/sales-reports`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sales-reports`.
  3. Open By product.
- **Expected UI Result:** Products with units, revenue, cost, margin.
- **Expected Backend / Data Result:** `salesByProduct` computes.
- **Cleanup:** None.

### REPORT-003 — Purchase reports load
- **Area:** Reports
- **Title:** Purchase reports render
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/purchasing-reports`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/purchasing-reports`.
  3. Open PO status.
- **Expected UI Result:** PO status breakdown shown.
- **Expected Backend / Data Result:** `poStatus` computes.
- **Cleanup:** None.

### REPORT-004 — Inventory reports load
- **Area:** Reports
- **Title:** Inventory reports render
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Route:** `/admin/inventory-reports`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/inventory-reports`.
  3. Open Valuation.
- **Expected UI Result:** Valuation shown.
- **Expected Backend / Data Result:** `valuation` computes.
- **Cleanup:** None.

### REPORT-005 — Report CSV export
- **Area:** Reports
- **Title:** Export a report to CSV
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/sales-reports`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sales-reports`.
  3. Click the CSV export button.
- **Expected UI Result:** A CSV file downloads.
- **Expected Backend / Data Result:** None (client-side download).
- **Cleanup:** None.

### REPORT-006 — Report date range filter
- **Area:** Reports
- **Title:** Date range filters reports
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/sales-reports`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sales-reports`.
  3. Change the From/To range.
- **Expected UI Result:** Report data updates.
- **Expected Backend / Data Result:** Server-side date filter.
- **Cleanup:** None.

### REPORT-007 — Report empty period
- **Area:** Reports
- **Title:** Empty period shows empty state
- **Priority:** P2
- **Test Type:** Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/sales-reports`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/sales-reports`.
  3. Set a date range with no sales.
- **Expected UI Result:** Empty state shown.
- **Expected Backend / Data Result:** No data.
- **Cleanup:** None.

### REPORT-008 — Report role access
- **Area:** Reports
- **Title:** Reports are role-gated
- **Priority:** P1
- **Test Type:** Permission, Security
- **Suitable for Playwright?:** Yes
- **Roles:** Staff
- **Route:** `/admin/sales-reports`
- **Steps:**
  1. Authenticate as staff.
  2. Navigate to `/admin/sales-reports`.
- **Expected UI Result:** Report data denied.
- **Expected Backend / Data Result:** `getSalesOverviewReport` requires `FINANCE_ROLES`.
- **Negative Assertions:** Staff must not see cost/margin reports.
- **Cleanup:** None.

---

## 26. Search / Filter / Pagination Test Cases

### SEARCH-001 — Orders server-side search and pagination
- **Area:** Search
- **Title:** Orders table searches and paginates server-side
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Search by order number.
  4. Change page.
- **Expected UI Result:** Results update; pagination works.
- **Expected Backend / Data Result:** Server-side `.ilike` + `.range`.
- **Cleanup:** None.
- **Notes for Playwright:** Orders is server-side paginated (see handoff).

### SEARCH-002 — Inventory movements server-side filters
- **Area:** Search
- **Title:** Movement log filters server-side
- **Priority:** P1
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/inventory`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/inventory`.
  3. Filter movements by type/date.
- **Expected UI Result:** Movements update.
- **Expected Backend / Data Result:** Server-side filters.
- **Cleanup:** None.

### SEARCH-003 — Products client-side search
- **Area:** Search
- **Title:** Products table searches client-side
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Search by name/SKU/barcode.
- **Expected UI Result:** Products filter.
- **Expected Backend / Data Result:** Client-side filtering.
- **Cleanup:** None.
- **Notes for Playwright:** Products is client-side filtered (see handoff).

### SEARCH-004 — Audit log pagination
- **Area:** Search
- **Title:** Audit log paginates
- **Priority:** P2
- **Test Type:** Happy Path
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/audit`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/audit`.
  3. Change page.
- **Expected UI Result:** Audit entries paginate.
- **Expected Backend / Data Result:** Server-side pagination.
- **Cleanup:** None.

---

## 27. Responsive / Mobile UI Test Cases

### RESP-001 — Admin drawer on mobile
- **Area:** Responsive
- **Title:** Admin sidebar collapses to a drawer on mobile
- **Priority:** P1
- **Test Type:** Responsive
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as owner.
  2. Set viewport to 375px.
  3. Navigate to `/admin`.
- **Expected UI Result:** Sidebar collapses; drawer opens via hamburger.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### RESP-002 — No horizontal overflow on admin pages
- **Area:** Responsive
- **Title:** Admin pages have no horizontal overflow at 375px
- **Priority:** P1
- **Test Type:** Responsive
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** All admin pages
- **Steps:**
  1. Authenticate as owner.
  2. Set viewport to 375px.
  3. Visit each admin page.
- **Expected UI Result:** No horizontal overflow.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### RESP-003 — POS mobile cart
- **Area:** Responsive
- **Title:** POS cart is a slide-in sheet on mobile
- **Priority:** P1
- **Test Type:** Responsive
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Set viewport to 375px.
  3. Navigate to `/admin/pos`.
  4. Add a product.
- **Expected UI Result:** Cart is a slide-in sheet with a sticky total.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### RESP-004 — POS payment sheet fits mobile
- **Area:** Responsive
- **Title:** Payment sheet fits the mobile viewport
- **Priority:** P1
- **Test Type:** Responsive
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Set viewport to 375px.
  3. Navigate to `/admin/pos`.
  4. Open the payment sheet.
- **Expected UI Result:** Payment sheet fits; Complete sale is reachable.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### RESP-005 — Storefront mobile navigation
- **Area:** Responsive
- **Title:** Storefront mobile menu works
- **Priority:** P1
- **Test Type:** Responsive
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>`
- **Steps:**
  1. As a guest, set viewport to 375px.
  2. Navigate to `/store/<slug>`.
  3. Open the mobile menu.
- **Expected UI Result:** Mobile menu opens; navigation works.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### RESP-006 — Modal layering/clipping
- **Area:** Responsive
- **Title:** Modals fit the screen without clipping
- **Priority:** P2
- **Test Type:** Responsive
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Set viewport to 375px.
  3. Navigate to `/admin/products`.
  4. Open the New product modal.
- **Expected UI Result:** Modal fits; no overflow/clipping.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

---

## 28. Loading / UI Stability Test Cases

### UX-001 — Async buttons show loading state
- **Area:** UX
- **Title:** Action buttons show a spinner and disable while busy
- **Priority:** P1
- **Test Type:** Regression, UX
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Trigger an async action (e.g. open session).
- **Expected UI Result:** Button shows a spinner and disables.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### UX-002 — Dashboard skeleton is stable
- **Area:** UX
- **Title:** Dashboard does not "dance" while loading
- **Priority:** P1
- **Test Type:** Regression, UX
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin`.
  3. Measure layout during load.
- **Expected UI Result:** Skeleton mirrors the real layout; no severe shift.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### UX-003 — Table rows persist during filter (keepPreviousData)
- **Area:** UX
- **Title:** Orders rows do not blank while filtering
- **Priority:** P2
- **Test Type:** Regression, UX
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/orders`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/orders`.
  3. Change a filter.
- **Expected UI Result:** Previous rows stay visible (dimmed) during fetch.
- **Expected Backend / Data Result:** `keepPreviousData`.
- **Cleanup:** None.

---

## 29. Error Handling Test Cases

### ERROR-001 — Failed server request shows an error
- **Area:** Error Handling
- **Title:** A failed mutation shows the real error message
- **Priority:** P1
- **Test Type:** Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products`.
  3. Trigger a failing mutation (e.g. duplicate SKU).
- **Expected UI Result:** The real backend error message is shown (via `getErrorMessage`).
- **Expected Backend / Data Result:** Error surfaced.
- **Negative Assertions:** No generic "Could not save…" fallback.
- **Cleanup:** None.

### ERROR-002 — Unauthorized action shows an error
- **Area:** Error Handling
- **Title:** Unauthorized mutation shows a permission error
- **Priority:** P1
- **Test Type:** Negative, Permission
- **Suitable for Playwright?:** Yes
- **Roles:** Cashier
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as cashier.
  2. Attempt a manager-only action.
- **Expected UI Result:** Permission error shown.
- **Expected Backend / Data Result:** Server guard throws.
- **Cleanup:** None.

### ERROR-003 — Not-found page
- **Area:** Error Handling
- **Title:** Unknown route shows a not-found state
- **Priority:** P2
- **Test Type:** Negative
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>/product/does-not-exist`
- **Steps:**
  1. As a guest, navigate to an invalid product.
- **Expected UI Result:** `Product not found` empty state.
- **Expected Backend / Data Result:** null result.
- **Cleanup:** None.

---

## 30. Concurrency / Duplication Test Cases

### CONC-001 — Double-click Complete Sale (see POS-030)
- **Area:** Concurrency
- **Title:** Duplicate sale prevention
- **Priority:** P0
- **Test Type:** Concurrency
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Notes:** Covered by POS-030. Verify exactly one order and one inventory deduction.

### CONC-002 — Duplicate sync (see OFFLINE-003)
- **Area:** Concurrency
- **Title:** Duplicate offline sync prevention
- **Priority:** P0
- **Test Type:** Concurrency
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Notes:** Covered by OFFLINE-003. `local_uuid` unique constraint prevents duplicates.

### CONC-003 — Duplicate payment verification (see PROOF-006)
- **Area:** Concurrency
- **Title:** Duplicate proof verification prevention
- **Priority:** P0
- **Test Type:** Concurrency
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Notes:** Covered by PROOF-006.

### CONC-004 — Duplicate stock receipt (see PURCHASE-010)
- **Area:** Concurrency
- **Title:** Duplicate goods receipt prevention
- **Priority:** P0
- **Test Type:** Concurrency
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Notes:** Covered by PURCHASE-010.

### CONC-005 — Simultaneous document number allocation
- **Area:** Concurrency
- **Title:** Concurrent sales get unique order numbers
- **Priority:** P0
- **Test Type:** Concurrency
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** An open cash session.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Fire two concurrent `posCheckout` requests.
- **Expected UI Result:** Two orders with unique order numbers.
- **Expected Backend / Data Result:** `next_document_number` RPC is atomic.
- **Negative Assertions:** No duplicate order numbers.
- **Cleanup:** Return/restock.
- **Notes for Playwright:** Use parallel API requests.

### CONC-006 — Concurrent stock deduction
- **Area:** Concurrency
- **Title:** Concurrent sales of the same variant deduct correctly
- **Priority:** P0
- **Test Type:** Concurrency, Inventory Integrity
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner
- **Preconditions:** A variant with 10 units.
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Fire two concurrent sales of 5 units each.
- **Expected UI Result:** Both succeed; balance = 0.
- **Expected Backend / Data Result:** `apply_inventory_delta` is atomic.
- **Negative Assertions:** No lost update.
- **Cleanup:** Adjust stock back.

---

## 31. Security / Abuse Test Cases

### SEC-001 — Manipulating IDs in URLs
- **Area:** Security
- **Title:** Another tenant's record ID is not accessible
- **Priority:** P0
- **Test Type:** Security, Tenant Isolation
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Owner (Shop A)
- **Steps:**
  1. Authenticate as Shop A owner.
  2. Attempt to access a Shop B record by ID.
- **Expected UI Result:** Not found / denied.
- **Expected Backend / Data Result:** All queries filter by `shop_id`.
- **Negative Assertions:** No IDOR.
- **Cleanup:** None.

### SEC-002 — Direct protected function call
- **Area:** Security
- **Title:** Direct server-function call without authorization fails
- **Priority:** P0
- **Test Type:** Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Steps:**
  1. As a guest, call a protected server function directly.
- **Expected UI Result:** Unauthorized error.
- **Expected Backend / Data Result:** `requireSupabaseAuth` middleware rejects.
- **Negative Assertions:** No unauthorized access.
- **Cleanup:** None.

### SEC-003 — XSS-like text in product/customer fields
- **Area:** Security
- **Title:** HTML/script in fields is not executed
- **Priority:** P1
- **Test Type:** Security
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Create a product with `<script>alert(1)</script>` in the name.
  3. View the product.
- **Expected UI Result:** The text is rendered as text, not executed.
- **Expected Backend / Data Result:** React escapes output.
- **Negative Assertions:** No XSS execution.
- **Cleanup:** Delete the QA product.

### SEC-004 — Malicious filename / invalid MIME upload
- **Area:** Security
- **Title:** Invalid upload MIME rejected
- **Priority:** P1
- **Test Type:** Security, Validation
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Attempt to upload a non-image file as a product image.
- **Expected UI Result:** Error `only JPG, PNG, WEBP, AVIF, GIF or SVG images are allowed.`
- **Expected Backend / Data Result:** `validateImageFile` rejects.
- **Cleanup:** None.

### SEC-005 — Storefront hidden product direct URL
- **Area:** Security
- **Title:** Inactive/internal product direct URL is not accessible
- **Priority:** P1
- **Test Type:** Security
- **Suitable for Playwright?:** Yes — requires API/database setup
- **Roles:** Guest
- **Preconditions:** An inactive or internal product exists.
- **Route:** `/store/<slug>/product/<slug>`
- **Steps:**
  1. As a guest, navigate to an inactive/internal product's URL.
- **Expected UI Result:** `Product not found`.
- **Expected Backend / Data Result:** `getStorefrontProduct` filters by `status = active`.
- **Negative Assertions:** Inactive product not accessible.
- **Cleanup:** None.

---

## 32. Accessibility Test Cases

### A11Y-001 — Form labels present
- **Area:** Accessibility
- **Title:** Key forms have accessible labels
- **Priority:** P2
- **Test Type:** Accessibility
- **Suitable for Playwright?:** Yes (can combine with axe)
- **Roles:** Owner
- **Route:** `/auth`, `/admin/products`, `/store/<slug>/checkout`
- **Steps:**
  1. Navigate to each form.
  2. Verify inputs have associated labels.
- **Expected UI Result:** Labels present.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### A11Y-002 — Button accessible names
- **Area:** Accessibility
- **Title:** Icon buttons have accessible names
- **Priority:** P2
- **Test Type:** Accessibility
- **Suitable for Playwright?:** Yes
- **Roles:** Guest
- **Route:** `/store/<slug>`
- **Steps:**
  1. As a guest, navigate to the storefront.
  2. Verify icon buttons have `aria-label`.
- **Expected UI Result:** Accessible names present.
- **Cleanup:** None.

### A11Y-003 — Modal focus and Escape
- **Area:** Accessibility
- **Title:** Modals trap focus and close on Escape
- **Priority:** P2
- **Test Type:** Accessibility
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Open the New product modal.
  3. Press Escape.
- **Expected UI Result:** Modal closes; focus returns.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### A11Y-004 — Keyboard navigation
- **Area:** Accessibility
- **Title:** Key flows are keyboard-accessible
- **Priority:** P2
- **Test Type:** Accessibility
- **Suitable for Playwright?:** Yes
- **Roles:** Owner
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate the admin via keyboard (Tab/Enter).
- **Expected UI Result:** Navigation works via keyboard.
- **Cleanup:** None.

---

## 33. Performance-Sensitive Cases

### PERF-001 — Large product search
- **Area:** Performance
- **Title:** Product search with 1000+ products
- **Priority:** P2
- **Test Type:** Performance
- **Suitable for Playwright?:** Partial (needs large dataset)
- **Roles:** Owner
- **Route:** `/admin/products`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/products` with 1000+ products.
  3. Search.
- **Expected UI Result:** Search is responsive.
- **Expected Backend / Data Result:** None.
- **Cleanup:** None.

### PERF-002 — POS product lookup
- **Area:** Performance
- **Title:** POS catalog loads and searches quickly
- **Priority:** P2
- **Test Type:** Performance
- **Suitable for Playwright?:** Partial
- **Roles:** Owner
- **Route:** `/admin/pos`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin/pos`.
  3. Measure catalog load and search.
- **Expected UI Result:** Responsive.
- **Cleanup:** None.

### PERF-003 — Dashboard load
- **Area:** Performance
- **Title:** Dashboard loads within a reasonable time
- **Priority:** P2
- **Test Type:** Performance
- **Suitable for Playwright?:** Partial
- **Roles:** Owner
- **Route:** `/admin`
- **Steps:**
  1. Authenticate as owner.
  2. Navigate to `/admin`.
  3. Measure load time.
- **Expected UI Result:** Loads within budget.
- **Cleanup:** None.

---

## 34. Playwright Automation Guidance

### 34.1 Suggested Project Structure

```
tests/e2e/
  auth/            # AUTH-* tests
  onboarding/      # ONBOARD-* tests
  tenant-isolation/# TENANT-* tests
  rbac/            # RBAC-* tests
  dashboard/       # DASH-* tests
  products/        # PRODUCT-* tests
  pos/             # POS-* tests
  offline/         # OFFLINE-* tests
  returns/         # RETURN-* tests
  customers/       # CUSTOMER-* tests
  inventory/       # INV-* tests
  purchasing/      # PURCHASE-* tests
  sales/           # SALES-* tests
  accounting/      # ACCOUNT-* tests
  expenses/        # EXPENSE-* tests
  storefront/      # STORE-* tests
  checkout/        # CHECKOUT-* tests
  payment-proof/   # PROOF-* tests
  domains/         # DOMAIN-* tests
  ai/              # AI-* tests
  data-manager/    # DM-* tests
  reports/         # REPORT-* tests
  security/        # SEC-* tests
  responsive/      # RESP-* tests
  accessibility/   # A11Y-* tests
```

### 34.2 Fixtures

Create fixtures for:
- `owner`, `admin`, `manager`, `cashier`, `accountant`, `inventory`, `staff` (authenticated storage states)
- `customer`, `guest` (storefront)
- `shopAOwner`, `shopBOwner` (tenant isolation)
- Seeded product, register, inventory, supplier

### 34.3 Storage State

Use role-specific Playwright storage states (e.g. `playwright/.auth/owner.json`) generated via real login flows. Do not commit session tokens. Global setup should fail clearly if required accounts are missing.

### 34.4 Data Isolation

Every automated run should create/use records prefixed with `QA-` or `QA-${RUN_ID}-`. Example: `QA-${RUN_ID}-Product`.

### 34.5 Cleanup

Clean up only the records the test created (prefix `QA-`). Do not reset the entire database. For destructive tests (Data Manager Reset), use an isolated QA shop.

### 34.6 Test Dependencies

Prefer independent tests. Use fixtures/setup or grouped serial tests only where the flow genuinely requires state progression (e.g. POS sale → return).

### 34.7 Database Assertions

Mark tests that need backend verification (see "Suitable for Playwright?" = "Yes — requires API/database setup"). Create DB assertion utilities carefully; keep tenant credentials/secrets out of browser code.

### 34.8 Screenshots / Trace / Video

- Screenshot on failure.
- Playwright trace on first retry.
- Video optional.

### 34.9 Parallelization

Do NOT run these groups concurrently (shared mutable resources):
- POS register/session tests (POS-002..006, POS-023..032)
- Inventory tests on the same SKU (POS-020..022, INV-*)
- Document sequence tests (POS-032, CONC-005)
- Offline sync tests (OFFLINE-*)
- Payment proof tests (PROOF-*)
- Supplier invoice/payment tests (PURCHASE-008..010)

Use serial mode or isolated fixtures for these.

---

## 35. Playwright Execution Matrix

| Suite | IDs | Browser | Role | DB Setup | Parallel? | Priority |
| --- | --- | --- | --- | --- | --- | --- |
| Auth | AUTH-* | Chromium | mixed | low | yes | P0/P1 |
| Onboarding | ONBOARD-* | Chromium | Owner | medium | yes | P0/P1 |
| Tenant | TENANT-* | Chromium | Owner/Admin | Shop A+B | limited | P0 |
| RBAC | RBAC-* | Chromium | all roles | low | yes | P0/P1 |
| Dashboard | DASH-* | Chromium | Owner | low | yes | P1/P2 |
| Products | PRODUCT-* | Chromium | Owner | medium | yes | P0/P1 |
| POS | POS-* | Chromium | Cashier/Manager | seeded register/product | no/limited | P0 |
| Offline | OFFLINE-* | Chromium | Owner | seeded catalog | no | P0/P1 |
| Returns | RETURN-* | Chromium | Owner | seeded sale | no | P0/P1 |
| Customers | CUSTOMER-* | Chromium | Owner | medium | yes | P1/P2 |
| Inventory | INV-* | Chromium | Owner/Inventory | medium | limited | P0/P1 |
| Purchasing | PURCHASE-* | Chromium | Owner | medium | limited | P0/P1 |
| Sales | SALES-* | Chromium | Owner | medium | yes | P0/P1 |
| Accounting | ACCOUNT-* | Chromium | Owner/Accountant | medium | yes | P0/P1 |
| Expenses | EXPENSE-* | Chromium | Owner | medium | yes | P1/P2 |
| Storefront | STORE-* | Chromium + Mobile | Guest/Customer | seeded catalog | yes | P1 |
| Checkout | CHECKOUT-* | Chromium | Guest | seeded catalog | yes | P0/P1 |
| Payment Proof | PROOF-* | Chromium | Guest/Owner | seeded order | no | P0 |
| Domains | DOMAIN-* | Chromium | Owner | low | yes | P0/P1 |
| AI | AI-* | Chromium | Owner | low | yes | P0/P1 |
| Data Manager | DM-* | Chromium | Owner/Admin | QA shop | no | P0 |
| Reports | REPORT-* | Chromium | Owner | medium | yes | P1/P2 |
| Security | SEC-* | Chromium | mixed | medium | yes | P0/P1 |
| Responsive | RESP-* | Chromium + Mobile | mixed | low | yes | P1/P2 |
| Accessibility | A11Y-* | Chromium | mixed | low | yes | P2 |

---

## 36. Test Case Coverage Summary

### 36.1 Totals

| Metric | Count |
| --- | --- |
| **Total test cases** | **~180** |
| P0 | ~60 |
| P1 | ~80 |
| P2 | ~35 |
| P3 | ~5 |
| Playwright automatable (Yes / Partial) | ~170 |
| Needs DB/API assertion | ~70 |
| Manual/external | ~10 |
| Security | ~25 |
| Tenant isolation | ~18 |
| Responsive | ~6 |
| Offline | ~7 |

### 36.2 Per Functional Area

| Area | IDs | Count |
| --- | --- | --- |
| Authentication | AUTH-001..010 | 10 |
| Onboarding | ONBOARD-001..004 | 4 |
| Multi-tenancy | TENANT-001..018 | 18 |
| RBAC | RBAC-001..021 | 21 |
| Dashboard | DASH-001..008 | 8 |
| Products/Catalog | PRODUCT-001..015 | 15 |
| POS | POS-001..035 | 35 |
| Offline POS | OFFLINE-001..007 | 7 |
| Returns | RETURN-001..006 | 6 |
| Customers/CRM | CUSTOMER-001..008 | 8 |
| Inventory | INV-001..010 | 10 |
| Purchasing | PURCHASE-001..010 | 10 |
| Sales/Orders | SALES-001..005 | 5 |
| Accounting | ACCOUNT-001..012 | 12 |
| Expenses | EXPENSE-001..002 | 2 |
| Storefront Admin | STORE-ADMIN-001..007 | 7 |
| Public Storefront | STORE-001..013 | 13 |
| Checkout | CHECKOUT-001..008 | 8 |
| Payment Proof | PROOF-001..010 | 10 |
| Custom Domains | DOMAIN-001..009 | 9 |
| AI Assistant | AI-001..007 | 7 |
| AI Salesman | AI-SALES-001..005 | 5 |
| Data Manager | DM-001..010 | 10 |
| Reports | REPORT-001..008 | 8 |
| Search/Filter | SEARCH-001..004 | 4 |
| Responsive | RESP-001..006 | 6 |
| UX Stability | UX-001..003 | 3 |
| Error Handling | ERROR-001..003 | 3 |
| Concurrency | CONC-001..006 | 6 |
| Security | SEC-001..005 | 5 |
| Accessibility | A11Y-001..004 | 4 |
| Performance | PERF-001..003 | 3 |

---

## 37. Code Review Findings

### 37.1 Fixed During This Review

#### FIX-001 — Discount not capped at line total
- **Severity:** P1 (financial integrity)
- **Affected file(s):** `src/lib/retail.server.ts` (`priceCart`)
- **Root cause:** A client-supplied `discount` larger than the line total (`unitPrice * quantity`) produced a negative `gross`, `net`, and `total`, allowing a cashier (or malicious authenticated user) to create a sale with a negative order total.
- **Change made:** Capped the discount at the line total: `discount = round(Math.min(Math.max(0, Number(item.discount) || 0), lineTotal))`. A discount equal to the line total (100% off) still works; anything larger is clamped to make the line free.
- **Regression test:** `POS-015` (line discount), `POS-031` (server recalculates price).

#### FIX-002 — Redundant ternary in commitImportFn
- **Severity:** P3 (code smell)
- **Affected file(s):** `src/lib/import.functions.ts`
- **Root cause:** `data.shopId ? context.userId : context.userId` always evaluates to `context.userId`.
- **Change made:** Simplified to `context.userId`.
- **Regression test:** `PRODUCT-011` (CSV import).

### 37.2 Open Findings

#### OPEN-001 — Client-side inventory adjustment race
- **Severity:** P2
- **Description:** `admin.inventory.tsx` `applyAdjustment` does a client-side read-modify-write on `inventory_balances` (insert movement, then read balance, then update). This bypasses the atomic `apply_inventory_delta` RPC used by server-side `postInventory`, so concurrent adjustments of the same variant could lose an update.
- **Affected files:** `src/routes/_authenticated/admin.inventory.tsx`
- **Impact:** Rare lost-update under concurrent adjustments.
- **Why not fixed:** The adjustment path is client-side via the RLS client; routing it through a server function is a larger change. Low frequency of concurrent adjustments.
- **Recommended approach:** Add a server function that uses `apply_inventory_delta` for adjustments.
- **Related tests:** `INV-002`, `INV-003`.

#### OPEN-002 — Products table is client-side filtered (no server pagination)
- **Severity:** P2 (performance)
- **Description:** The Products table loads the full product set and filters client-side. With 1000+ products this is heavy.
- **Affected files:** `src/routes/_authenticated/admin.products.tsx`
- **Impact:** Performance degradation with large catalogs.
- **Why not fixed:** Documented in handoff as "Deferred — High Impact" (needs RPC/search view for cross-child search).
- **Recommended approach:** Server-side pagination with a search view.
- **Related tests:** `PRODUCT-010`, `PERF-001`.

#### OPEN-003 — Multi-table sale posting uses best-effort rollback, not a single DB transaction
- **Severity:** P2
- **Description:** `recordPosSale` uses the atomic `record_pos_sale` RPC for the main write, but some follow-ups (low-stock alerts, events) run post-commit. The RPC itself is atomic; the follow-ups are fire-and-forget.
- **Affected files:** `src/lib/pos.server.ts`
- **Impact:** Low — the RPC is atomic; follow-ups are non-critical.
- **Why not fixed:** The RPC already provides atomicity for the critical path.
- **Recommended approach:** None needed for the critical path.
- **Related tests:** `POS-032`.

#### OPEN-004 — POS catalog limited to 500 variants
- **Severity:** P3
- **Description:** `posCatalog` limits to 500 variants.
- **Affected files:** `src/lib/pos.functions.ts`
- **Impact:** Large catalogs may not fully load in POS.
- **Why not fixed:** Reasonable default; larger catalogs need pagination/search.
- **Recommended approach:** Server-side search/pagination for POS catalog.
- **Related tests:** `PERF-002`.

---

## 38. Documentation Drift Notes

- The `README.md` and `features.md` accurately describe the current implementation. No significant drift found.
- The `handoff.md` "Deferred — High Impact" section accurately lists the Products table and accounting reports as client-side/deferred.
- The existing Playwright config (`playwright.config.ts`) targets `http://localhost:8082` by default and uses `channel: "msedge"`. For live testing against `https://goretailos.com`, set `PW_BASE_URL=https://goretailos.com`.
- The existing `auth.setup.ts` already handles the server-rendered `/auth` click-swallow issue (retry loop). Reuse this pattern.

---

*End of TEST_CASES.md*
---
# Execution Status (merged from TEST_EXECUTION_STATUS.md, 2026-09-29)

# GoRetailOS — Test Execution Status

This file maps every `TEST_CASES.md` ID to its implementation/execution status.

Legend:
- ✅ Automated + Passed
- ❌ Automated + Failed
- ⚠️ Automated + Known Issue
- ⏭ Not applicable
- 🧪 Manual/External
- ⏳ Not yet implemented

## Summary

| Metric | Count |
| --- | --- |
| Total master cases (TEST_CASES.md) | ~180 |
| Automated + Passed | 39 |
| Automated + Failed | 0 |
| Skipped (staff auth, no PW_EMAIL) | 1 |
| Not yet implemented | ~140 |
| Manual/External | ~10 |

> **Note:** The remaining suites (PRODUCT, CUSTOMER, INV, RETURN, PURCHASE, ACCOUNT, CHECKOUT, PROOF, OFFLINE, AI, DM, DOMAIN, REPORT, etc.) require a dedicated QA tenant (for mutation-heavy tests), additional persona credentials (for full RBAC), or a second shop (for full tenant isolation). The local dev server shares the production database, so mutation-heavy tests are deferred to avoid modifying production data.

## Authentication (AUTH)

| ID | Status | Notes |
| --- | --- | --- |
| AUTH-001 | ✅ | owner-auth.spec.ts |
| AUTH-002 | ✅ | visitor-auth.spec.ts |
| AUTH-003 | ✅ | visitor-auth.spec.ts |
| AUTH-004 | ✅ | visitor-auth.spec.ts |
| AUTH-005 | ⏳ | Requires profile menu sign-out flow |
| AUTH-006 | ✅ | visitor-auth.spec.ts |
| AUTH-007 | ✅ | owner-auth.spec.ts |
| AUTH-008 | ⏳ | Requires email confirmation handling |
| AUTH-009 | ✅ | visitor-auth.spec.ts |
| AUTH-010 | ✅ | visitor-auth.spec.ts |

## Onboarding (ONBOARD)

| ID | Status | Notes |
| --- | --- | --- |
| ONBOARD-001 | ⏳ | Requires a fresh user with no shops |
| ONBOARD-002 | ⏳ | Requires duplicate slug |
| ONBOARD-003 | ⏳ | Requires onboarding form |
| ONBOARD-004 | ⏳ | Requires onboarding form |

## Multi-Tenancy (TENANT)

| ID | Status | Notes |
| --- | --- | --- |
| TENANT-001..015 | ⏳ | Requires a second QA shop (Shop B) |
| TENANT-016 | ✅ | owner-tenant.spec.ts (shop switcher) |
| TENANT-017 | ✅ | owner-tenant.spec.ts (storefront scoping) |
| TENANT-018 | ✅ | owner-tenant.spec.ts (cost price) |

## RBAC (RBAC)

| ID | Status | Notes |
| --- | --- | --- |
| RBAC-001 | ✅ | owner-rbac.spec.ts (owner sees all nav) |
| RBAC-002 | ✅ | owner-rbac.spec.ts (unauthenticated server call rejected) |
| RBAC-003..008 | ⏳ | Requires additional personas |
| RBAC-009 | ✅ | owner-rbac.spec.ts (owner Data Manager) |
| RBAC-010..011 | ⏳ | Requires additional personas |
| RBAC-012 | ✅ | owner-rbac.spec.ts (owner audit) |
| RBAC-013 | ✅ | owner-rbac.spec.ts (owner team) |
| RBAC-014..021 | ⏳ | Requires additional personas |

## Dashboard (DASH)

| ID | Status | Notes |
| --- | --- | --- |
| DASH-001..008 | ⏳ | Not yet implemented |

## Products/Catalog (PRODUCT)

| ID | Status | Notes |
| --- | --- | --- |
| PRODUCT-001..015 | ⏳ | Not yet implemented (requires product CRUD) |

## POS (POS)

| ID | Status | Notes |
| --- | --- | --- |
| POS-001..030 | ⏳ | Not yet implemented |
| POS-031 | ✅ | owner-pos-financial.spec.ts (discount cap regression) |
| POS-025 | ✅ | owner-pos-financial.spec.ts (under-allocation rejected) |
| POS-032..035 | ⏳ | Not yet implemented |

## Offline POS (OFFLINE)

| ID | Status | Notes |
| --- | --- | --- |
| OFFLINE-001..007 | ⏳ | Not yet implemented (requires context.setOffline) |

## Returns (RETURN)

| ID | Status | Notes |
| --- | --- | --- |
| RETURN-001..006 | ⏳ | Not yet implemented |

## Customers/CRM (CUSTOMER)

| ID | Status | Notes |
| --- | --- | --- |
| CUSTOMER-001..008 | ⏳ | Not yet implemented |

## Inventory (INV)

| ID | Status | Notes |
| --- | --- | --- |
| INV-001..010 | ⏳ | Not yet implemented |

## Purchasing (PURCHASE)

| ID | Status | Notes |
| --- | --- | --- |
| PURCHASE-001..010 | ⏳ | Not yet implemented |

## Sales/Orders (SALES)

| ID | Status | Notes |
| --- | --- | --- |
| SALES-001..005 | ⏳ | Not yet implemented |

## Accounting (ACCOUNT)

| ID | Status | Notes |
| --- | --- | --- |
| ACCOUNT-001..012 | ⏳ | Not yet implemented |

## Expenses (EXPENSE)

| ID | Status | Notes |
| --- | --- | --- |
| EXPENSE-001..002 | ⏳ | Not yet implemented |

## Storefront Admin (STORE-ADMIN)

| ID | Status | Notes |
| --- | --- | --- |
| STORE-ADMIN-001..007 | ⏳ | Not yet implemented |

## Public Storefront (STORE)

| ID | Status | Notes |
| --- | --- | --- |
| STORE-003 | ✅ | visitor-storefront-cart.spec.ts |
| STORE-005 | ✅ | visitor-storefront-cart.spec.ts |
| STORE-006 | ✅ | visitor-storefront-cart.spec.ts |
| STORE-007 | ✅ | visitor-storefront-cart.spec.ts |
| STORE-008 | ✅ | visitor-storefront-cart.spec.ts |
| STORE-009 | ✅ | visitor-storefront-security.spec.ts |
| STORE-010 | ✅ | visitor-storefront-security.spec.ts |
| STORE-011 | ✅ | visitor-storefront-security.spec.ts |
| STORE-012 | ⏳ | Not yet implemented |
| STORE-013 | ✅ | Existing storefront-responsive.spec.ts |

## Checkout (CHECKOUT)

| ID | Status | Notes |
| --- | --- | --- |
| CHECKOUT-001..008 | ⏳ | Not yet implemented |

## Payment Proof (PROOF)

| ID | Status | Notes |
| --- | --- | --- |
| PROOF-001..010 | ⏳ | Not yet implemented |

## Custom Domains (DOMAIN)

| ID | Status | Notes |
| --- | --- | --- |
| DOMAIN-001..009 | ⏳ | Not yet implemented |

## AI (AI)

| ID | Status | Notes |
| --- | --- | --- |
| AI-001..007 | ⏳ | Not yet implemented |

## AI Salesman (AI-SALES)

| ID | Status | Notes |
| --- | --- | --- |
| AI-SALES-001..005 | ⏳ | Not yet implemented |

## Data Manager (DM)

| ID | Status | Notes |
| --- | --- | --- |
| DM-001..010 | ⏳ | Not yet implemented |

## Reports (REPORT)

| ID | Status | Notes |
| --- | --- | --- |
| REPORT-001..008 | ⏳ | Not yet implemented |

## Search/Filter (SEARCH)

| ID | Status | Notes |
| --- | --- | --- |
| SEARCH-001..004 | ⏳ | Not yet implemented |

## Responsive (RESP)

| ID | Status | Notes |
| --- | --- | --- |
| RESP-001..006 | ⏳ | Not yet implemented |

## UX Stability (UX)

| ID | Status | Notes |
| --- | --- | --- |
| UX-001..003 | ⏳ | Not yet implemented |

## Error Handling (ERROR)

| ID | Status | Notes |
| --- | --- | --- |
| ERROR-001..003 | ⏳ | Not yet implemented |

## Concurrency (CONC)

| ID | Status | Notes |
| --- | --- | --- |
| CONC-001..006 | ⏳ | Not yet implemented |

## Security (SEC)

| ID | Status | Notes |
| --- | --- | --- |
| SEC-001..005 | ⏳ | Not yet implemented |

## Accessibility (A11Y)

| ID | Status | Notes |
| --- | --- | --- |
| A11Y-001..004 | ⏳ | Not yet implemented |

## Performance (PERF)

| ID | Status | Notes |
| --- | --- | --- |
| PERF-001..003 | ⏳ | Not yet implemented |

## Smoke (SMOKE)

| ID | Status | Notes |
| --- | --- | --- |
| SMOKE-001..010 | ✅ | owner-smoke.spec.ts |
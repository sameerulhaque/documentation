# RetailOS — Detailed User Manual (Forms, Fields & Connections)

> **Companion to the [Usage Guide](/usage).** The usage guide explains *what the system does and how the flows work*; **this manual is the field-level reference:** for every form it lists each input — name, type, whether it's required, its default, its validation — and, crucially, **how that form connects to the others** (what it looks up, what it feeds, and what happens on submit). This manual is served at [`/manual`](/manual).
>
> **Base URL:** `https://goretailos.com`. Admin paths need a signed-in staff session; storefront paths carry a shop slug (`/store/<shopSlug>/…`, demo `trendies-store`).

---

## How to read this manual

> 💡 **Prefer learning inside the app?** Open **Quick Tours** (`/admin/tours`, under
> Administration in the sidebar): a guided walkthrough of the whole system plus a detailed
> step-by-step tour for every module your shop has enabled — POS, Restaurant, E-commerce,
> Inventory, Purchasing, Accounting, HR & Payroll, Growth and the AI assistant. Each step takes
> you to the real screen and tells you exactly what to do there — tours never change your data,
> and your progress is saved so you can resume any time.

Each form is documented as:

- **Purpose / Access / Backend** — what it's for, which roles may use it, and the server function + database table(s) behind it (so developers can trace it).
- A **Fields** table with these columns:

  | Column | Meaning |
  | --- | --- |
  | **UI label** | What the user sees on screen. |
  | **Field key** | The underlying form/database field name. |
  | **Type** | text, number, money, integer, select, multi-select, toggle, date, file, textarea, color, computed. |
  | **Req** | Required (shown as ● or "Yes") · optional (○ or "No") · `auto` = set by the system, not typed by the user. |
  | **Default** | Pre-filled value, if any. |
  | **Validation / options** | Constraints, or the choice list for selects. |
  | **Notes** | Anything else worth knowing. |

- A **Connections** block:
  - **Selects / references** — other forms this one *reads from* (foreign keys). Example: a Product **selects** a Category.
  - **Feeds / used by** — forms and screens that *consume* this record downstream.
  - **Cross-form effects** — what submitting does elsewhere (stock movements, ledger postings, sequence numbers).

> **Convention used everywhere:** a field ending in `_id` is a pointer to another form's record. The manual names the human form (e.g. "Category") and the pointer (`category_id`) together.

---

## The tenant model (read this first)

Everything hangs off one root record: the **shop**. **Every** table in the system carries a `shop_id`, and the server always filters by the shop you have active in the header switcher. A single login can be a member of several shops (see *Team & Roles*) and switch between them; data never crosses between shops. Branch-bound staff are further limited to one **branch** within a shop.

**Header controls.** The top bar carries the **organization switcher** (your own shops appear under *My organizations*; platform-level accounts also see an *All organizations (platform)* group with search), a **branch** switcher, a notifications bell, a language selector, and a **Toggle theme** button that flips the admin between **light and dark** appearance (saved per browser). Switching organizations swaps *everything* at once — data, currency and timezone — with no cross-shop leakage; your newest organization starts completely empty.

Because of this, `shop_id` (and often `branch_id`) is an implicit, `auto` field on **every** form below — set from your session/active shop, never typed — so the per-form tables don't repeat it unless it behaves unusually.

### Data model at a glance

The forms wire together along these spines (arrow = "points at / looks up"):

**Catalog spine**
- **Product** → looks up **Category** (`category_id`, a self-nesting tree via `parent_id`), **Brand** (`brand_id`), **Tax rate** (`tax_rate_id`).
- **Product** → has many **Variants** (`product_variants`, the sellable unit). Each Variant fans out to **Barcodes**, **Prices**, **Variant costs** (private, never public), **Product images**, and **Inventory balances** (one per stock **Location**).

**Selling spine**
- **Order** (`order_channel` = pos / online / ai) → **Order items** (→ Variant) + **Payments** (→ Payment method) + **Receipts**; optionally → **Customer** (`customer_id`).
- **Return** (→ Order) → **Return items** (→ Order item) → **Refunds**.
- **POS**: **Register** → **Cash session** → **Cash movements**; **Register users** grant till access.

**Inventory spine**
- **Inventory movement** (`movement_type`) is the ledger of stock; it updates **Inventory balances** (Variant × Location).
- **Transfer** → **Transfer items**, moving stock between **Locations** (`from_location_id` → `to_location_id`) on receipt.
- **Stock count** → **Stock count items** post adjustments.

**Purchasing spine**
- **Supplier** → **Purchase order** → **PO items** (→ Variant); **Goods receipt** (→ PO) → **Receipt items**; **Supplier invoice** (→ PO); **Supplier payment** (→ Supplier / Invoice).

**Accounting spine**
- Every posting writes a balanced **Journal entry** → **Journal entry lines** (→ **Account**; Accounts form a tree via `parent_id`, typed by `account_type`). **Expenses** (→ Expense category, Account). **Payment methods**, **Bank accounts**, **Cash accounts**.

**Storefront + AI**
- **Cart** → **Cart items** (→ Variant). **Customer** → **Customer addresses**, **Wishlists**. Store look/behaviour: **Storefront settings / themes / pages / navigation / banners / sections / policies**.
- **AI permissions** gate assistant actions; **Knowledge documents** → **Knowledge chunks**; **Conversations** → **Messages**, **Tool calls**, **Action approvals**, **Alerts**.

A full table-by-table foreign-key index is in **Appendix B**.

---

## Contents

1. [Accounts, Team & Shop Settings](#accounts-team--shop-settings)
2. [Catalog](#catalog)
3. [Inventory](#inventory)
4. [Purchasing & Accounting](#purchasing--accounting)
5. [Sales — POS, Registers, Orders & Sync](#sales--pos-registers-orders--sync)
6. [Storefront Editor & AI (Admin)](#storefront-editor--ai-admin)
7. [Storefront — Customer Forms](#storefront--customer-forms)
8. [Reports](#reports)
9. [Promotions, Loyalty & Marketing](#promotions-loyalty--marketing)
10. [Manufacturing (BOM & costing)](#manufacturing-bom--costing)
11. [Restaurant](#restaurant)
12. [CRM, Segments & Analytics](#crm-segments--analytics)
13. [HR & Payroll](#hr--payroll)
14. [Business Intelligence](#business-intelligence)
15. [Advanced Procurement](#advanced-procurement)
- [Appendix A — Enumerations (shared choice lists)](#appendix-a--enumerations)
- [Appendix B — Tables ↔ forms index](#appendix-b--tables--forms-index)

---

## Accounts, Team & Shop Settings

### Sign in / Sign up — `/auth` (`?mode=signup`)
- **Purpose:** Staff authentication and new-account registration for the admin portal.
- **Access:** Public (unauthenticated). Already-signed-in users are redirected to `/admin`.
- **Backend:** Supabase Auth client SDK (no server fn / zod) — `supabase.auth.signUp` / `supabase.auth.signInWithPassword` → `auth.users` (a DB trigger mirrors into `profiles`). `mode=signup` search param preselects the sign-up view.

**Fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Full name | fullName | text | ● (signup only) | "" | — | Passed as `options.data.full_name` → user metadata → `profiles.full_name`. Hidden in sign-in mode. |
| Email | email | email | ● | "" | trimmed | Used for both signup and signin |
| Password | password | password | ● | "" | Supabase policy (min 6) | — |
| — | mode | search param | — | (signin) | `"signup"` toggles view | Also toggled by "Create a new account" link |

**Connections**
- **Feeds / used by:** Session drives all `_authenticated/*` routes; `full_name`/`email` surface in *Team → Members*. New signup with `emailRedirectTo=/admin`.
- **Cross-form effects:** Successful auth navigates to `/admin` (which routes to *Onboarding* if the user has no shop). Signup without a session shows "check your email to confirm."

![Staff sign-in](docs/screenshots/auth/staff-sign-in.png)

After signing in you land on the **admin dashboard** (`/admin`) — KPI cards with trends, a 14-day sales chart, category mix, profitability, AI insights and an alert centre.

![Admin dashboard](docs/screenshots/admin/dashboard.png)

---

### Forgot password — `/auth` (inline action)
- **Purpose:** Trigger a password-reset email.
- **Access:** Public (sign-in view only).
- **Backend:** `supabase.auth.resetPasswordForEmail(email, { redirectTo: /reset-password?next=%2Fadmin })` (client SDK).

**Fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Email | email | email | ● | "" | non-empty (reuses the sign-in Email input) | Always shows generic "if that email has an account…" message |

**Connections**
- **Cross-form effects:** Email link lands on *Reset password* with `next=/admin`.

---

### Reset password — `/reset-password` (`?next=<relative path>`)
- **Purpose:** Set a new password from a recovery link (used by both staff and storefront customers).
- **Access:** Public, but the Update button is enabled only once a recovery/`SIGNED_IN` session is present.
- **Backend:** `supabase.auth.updateUser({ password })` (client SDK) → `auth.users`.

**Fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| New password | password | password | ● | "" | min 6 chars | — |
| Confirm new password | confirm | password | ● | "" | must equal `password` | — |
| — | next | search param | — | `/admin` | must start with `/` and not `//` (same-origin only) | Redirect target after success |

**Connections**
- **Cross-form effects:** On success redirects to `next` (default `/admin`) after ~1.2s.

---

### Onboarding — create shop — `/onboarding` (auth-gated)
- **Purpose:** Provision a new tenant (shop) for the signed-in user, who becomes owner.
- **Access:** Any authenticated user (creator is inserted as `owner`).
- **Sign out:** A **Sign out** button in the header ends the current session and returns to `/auth` — use it if you signed in with the wrong account before creating a shop.
- **Backend:** `createShop` (`src/lib/shop.functions.ts`) → `provisionShop` (`src/lib/shop.server.ts`) → tables `shops`, `shop_users`, and many child tables (see cross-form effects).

**Fields** (authoritative = `createShop` zod schema)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Shop name | name | text | ● | "" | 2–80 chars (button disabled < 2) | — |
| Store address (URL slug) | slug | text | ● | auto from name | 2–60 chars; server re-slugifies (`[^a-z0-9-]`→`-`) & checks uniqueness | Client defaults to `name` lowercased-hyphenated |
| Currency | currency_code | select | ● | "USD" | 2–5 chars; USD/PKR/QAR/AED/SAR/INR/EUR/GBP | — |
| — | currency_symbol | text | ● (auto) | "$" | 1–5 chars | `auto` — derived client-side from selected currency |
| Timezone | timezone | select | ● | "UTC" | UTC/Asia/Karachi/Riyadh/Qatar/Dubai/Kolkata/Europe/London/America/New_York | — |
| Default language | default_locale | select | ● | "en" | enum en/ar/ur/hi | From `SUPPORTED_LOCALES` |
| Storefront languages | supported_locales | checkbox[] | ● | ["en"] | array of en/ar/ur/hi, min 1 | Falls back to `["en"]` if none checked |
| Phone | phone | text | ○ | "" | ≤ 40 chars | — |
| Email | email | text | ○ | "" | ≤ 120 chars (not validated as email) | — |
| Address | address | text | ○ | "" | ≤ 240 chars | — |
| — | created_by | uuid | auto | `context.userId` | — | Server-set |
| — | settings | jsonb | auto | `DEFAULT_SHOP_SETTINGS` | — | Large default object (tax/receipt/inventory/pricing/etc.) |

**Connections**
- **Feeds / used by:** Creates the tenant consumed by every admin/POS/storefront route; owner membership drives `getMyShops` and the role matrix.
- **Cross-form effects (all in `provisionShop`, one batch of inserts):**
  - `shops` (with `settings = DEFAULT_SHOP_SETTINGS`), `shop_users` (creator → `owner`)
  - `branches`: "Main Branch" (`MAIN`, `is_default`, default business_hours)
  - `registers`: "Register 1" (`REG1`); `inventory_locations`: "Main Store" (`is_default`)
  - `payment_methods`: Cash / Card / Bank Transfer / Store Credit (disabled) / Cash on Delivery
  - `tax_rates`: "Standard VAT" rate 0 (`is_default`)
  - `accounts`: 14-line chart of accounts (`is_system`)
  - `document_sequences`: 8 doc types (order/receipt/invoice/purchase_order/return/credit_note/debit_note/goods_receipt)
  - `ai_permissions`: 19 capability rows; `storefront_settings` (1 row); `storefront_navigation` (5); `storefront_policies` (4); `expense_categories` (5)

---

### Accept invitation — `/invite/$token`
- **Purpose:** Let an invited user join a shop's team.
- **Access:** Preview is public; accepting requires being signed in with the invited email.
- **Backend:** `getInvitationPreview` (unauth) and `acceptStaffInvitation` (auth) (`src/lib/team.functions.ts`) → `team.server.ts` → tables `staff_invitations` (read + update) and `shop_users` (upsert).

**Fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| — | token | URL param | ● | — | ≥ 10 chars | Route param `$token` |
| (Accept invitation button) | — | action | — | — | requires signed-in email == invite email (case-insensitive) | Disabled/blocked if revoked, accepted, expired, or email mismatch |

Preview returns (read-only, `auto`): `status`, `expired`, `role`, `email`, `shopName`.

**Connections**
- **Selects / references:** Resolves shop name via `staff_invitations.shop_id → shops(name)`.
- **Cross-form effects on accept:** upsert into `shop_users` (`shop_id`, `user_id`, `role`, `branch_id`, `is_active=true`, onConflict `shop_id,user_id`); set `staff_invitations.status='accepted'`, `accepted_at`, `accepted_by`; audit log; navigates to `/admin`.

---

### Invite a staff member — `/admin/team`

![Team members](docs/screenshots/admin/team-members.png)

- **Purpose:** Send a staff invitation and generate a shareable invite link.
- **Access:** `owner`, `admin` — page requires `can("settings")`; server enforces `requireRole(..., ADMINS)`.
- **Backend:** `inviteStaffMember` (`src/lib/team.functions.ts`) → `inviteStaff` (`team.server.ts`) → table `staff_invitations`.

**Fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Email | email | email | ● | "" | `z.string().email()`; lowercased on insert | — |
| Role | role | select | ● | "staff" | enum owner/admin/manager/cashier/accountant/inventory/staff | — |
| Branch | branchId | select | ○ | "" → null | uuid nullable; "Any branch" = null | Options from *Branches* (shop-scoped) |
| — | shopId | uuid | auto | active shop | — | From `useAdminShop` |
| — | origin | url | auto | `window.location.origin` | — | Used to build invite link |
| — | token | text | auto | `gen_random_bytes(24)` hex | — | DB-generated |
| — | status | text | auto | "pending" | pending/accepted/revoked/expired | — |
| — | expires_at | timestamptz | auto | now + 14 days | — | — |
| — | invited_by | uuid | auto | `context.userId` | — | — |

**Connections**
- **Selects / references:** Branch (from *Branches*) via `branch_id`.
- **Feeds / used by:** Row appears in *Invitations* list; link consumed by *Accept invitation*. Returns `{ link: <origin>/invite/<token> }` shown with a Copy button.
- **Cross-form effects:** Insert into `staff_invitations`; audit `staff.invite`.

---

### Revoke invitation — `/admin/team` (Invitations list)
- **Purpose:** Cancel a pending invitation.
- **Access:** `owner`, `admin`.
- **Backend:** `revokeStaffInvitation` → `revokeInvitation` → `staff_invitations` (set `status='revoked'`).

**Fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (Revoke button) | invitationId | uuid | ● | — | uuid | Only shown when `status='pending'` |
| — | shopId | uuid | auto | active shop | — | — |

---

### Edit / deactivate member — `/admin/team` (Members list)
- **Purpose:** Change a member's role/branch or (de)activate them.
- **Access:** `owner`, `admin`.
- **Backend:** `updateTeamMember` (`src/lib/team.functions.ts`) → `updateMember` (`team.server.ts`) → table `shop_users`.

**Fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Role (inline select) | role | select | ○ | member's current | enum owner/admin/manager/cashier/accountant/inventory/staff | Saves on change |
| Branch (inline select) | branchId | select | ○ | member's current | uuid nullable; "Any branch"=null | Options from *Branches* |
| Deactivate / Reactivate | isActive | boolean | ○ | member's current | toggles `is_active` | Button label flips on state |
| — | memberId | uuid | ● | — | the `shop_users.id` | — |
| — | shopId | uuid | auto | active shop | — | — |

**Connections**
- **Cross-form effects:** Updates `shop_users`; audit `staff.member.update`. **Guard:** cannot demote or deactivate the last active `owner` ("must always have at least one active owner").

---

## Shop Settings — `/admin/settings`

![Shop settings](docs/screenshots/settings/shop-settings.png)

- **Backend (all sections except Tax rates & Branches):** `updateShopSection` (`src/lib/shop.functions.ts`) writes `shops` columns (the `shop` patch) and/or merges into `shops.settings[<section>]` jsonb (the `settingsPatch`). Section role gate enforced server-side via `has_shop_role`; the write uses the service-role client *after* the check.
- **Access per section:** Business, Localization, Tax, Document numbering, Customers & credit → `owner`, `admin`. Receipts, POS, Inventory, Pricing, Purchasing, Business hours, Notifications → `owner`, `admin`, `manager`. Lower roles see the section read-only.

### Business profile — settings section `business`
Writes `shops` columns + `settings.business.tax_id`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Shop name | name | text (shop col) | ● | `shops.name` | trimmed, non-empty | — |
| Store address (slug) | slug | text (shop col) | — | `shops.slug` | read-only | "Contact support to change" |
| Legal / registered name | legal_name | text (shop col) | ○ | "" → null | — | — |
| Tax ID / registration number | tax_id | text (`settings.business`) | ○ | "" | — | Only settings-jsonb field here |
| Email | email | email (shop col) | ○ | "" → null | — | — |
| Phone | phone | text (shop col) | ○ | "" → null | — | — |
| Business logo | logo_url | image upload (shop col) | ○ | "" → null | via `AdminImageUpload` (folder `branding`) | Used on receipts/invoices/storefront |
| Address | address | textarea (shop col) | ○ | "" → null | — | — |
| Timezone | timezone | text (shop col) | ● | `shops.timezone` | free-text | — |

### Localization — settings section `localization` (writes `shops` columns)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Default language | default_locale | select | ● | `shops.default_locale` | en/ar/ur/hi | Drives RTL |
| Currency code | currency_code | text | ● | `shops.currency_code` | uppercased | — |
| Currency symbol | currency_symbol | text | ● | `shops.currency_symbol` | — | — |
| Symbol position | currency_position | select | ● | before | before / after | — |
| Decimal places | decimal_places | number | ● | 2 | 0–4 | — |
| Decimal separator | decimal_separator | text | ● | "." | — | — |
| Thousand separator | thousand_separator | text | ● | "," | — | — |
| Date format | date_format | text | ● | "dd/MM/yyyy" | — | — |
| Time format | time_format | select | ● | "HH:mm" | 24h / 12h | — |

### Tax — settings section `tax` (+ inline `tax_rates` editing)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Tax registered | enabled | toggle | ○ | true | — | — |
| Prices include tax | inclusive | toggle | ○ | false | — | — |
| Tax registration number | registration_number | text | ○ | "" | — | — |
| Rounding rule | rounding | select | ○ | "nearest_0.05" | none / nearest_0.05 / nearest_0.10 / nearest_1 | — |

*Tax rates sub-table (writes `tax_rates` directly):* `rate` (number, saved on blur), `is_default` (radio — one default).

### Receipts — settings section `receipt`

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Header text | header | textarea | ○ | "" | — | — |
| Footer text | footer | textarea | ○ | "" | — | — |
| Show tax breakdown | show_tax_number | toggle | ○ | true | — | — |
| Show logo on receipt | show_logo | toggle | ○ | true | — | — |
| Roll width | roll_width | select | ○ | "80mm" | 58mm / 80mm | — |
| Reprint policy | reprint_policy | select | ○ | "allowed" | allowed / manager_approval / blocked | — |

*Printer (this device only, saved to localStorage — not DB):* Printer type (browser / escpos-bridge), Paper width, Bridge URL, Printer name, Auto-cut, Kick cash drawer.

### Point of sale — settings section `pos`

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Default branch | default_branch_id | select | ○ | "" | from *Branches* | Clears register on change |
| Default register | default_register_id | select | ○ | "" | from *Registers* filtered by branch | — |
| Allow line-item discounts | allow_line_discounts | toggle | ○ | true | — | — |
| Maximum discount % | max_discount_percent | number | ○ | 20 | 0–100 | Cashier ceiling |
| Require customer on sale | require_customer | toggle | ○ | false | — | — |
| Cash rounding | cash_rounding | select | ○ | "none" | none / nearest_0.05 / nearest_0.10 | — |
| Offline mode | offline_mode_enabled | toggle | ○ | true | — | — |

### Inventory — settings section `inventory`

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Allow negative stock | allow_negative_stock | toggle | ○ | false | — | Shop-wide switch; still governs the online-store checkout |
| Who can oversell (negative stock) | oversell_roles | multi-select (roles) | ○ | owner/admin/manager | role checklist | Only these roles may complete a **till** sale that drives stock below zero (enforced client- and server-side); owner/admin always included |
| Default location | default_location_id | select | ○ | default location | from *Inventory locations* | — |
| Low-stock threshold (default) | low_stock_threshold | number | ○ | 5 | ≥ 0 | — |
| Reorder point | reorder_point | number | ○ | 10 | ≥ 0 | — |
| Reorder quantity | reorder_quantity | number | ○ | 20 | ≥ 0 | — |

### Delivery charges — settings section `delivery`
- **Purpose:** Storefront delivery pricing — a **default policy** plus optional **per-zone** overrides resolved **exact city → country-wide → default**. Charges are computed server-side at checkout and set the order's `shipping_total`. **Backend:** `retail.server.ts` (`resolveDeliveryPolicy` / `computeDeliveryCharge` / `deliveryFreeThreshold`), read via `getShopDeliverySettings`; stored under `shops.settings.delivery`.

| UI label | Field key | Type | Req | Default | Notes |
|---|---|---|---|---|---|
| Enable delivery | `enabled` | toggle | ○ | false | off → checkout shows no delivery step |
| Charge mode | `charge_mode` | select | ○ | flat | **flat** (one amount) or **slabs** (tiered by order value) |
| Flat charge | `flat_charge` | number | ○ (flat) | 0 | delivery fee in flat mode |
| Slabs | `slabs[]` | rows | ○ (slabs) | — | tiered charge bands by order value |
| Free delivery above | `free_delivery_threshold` | number | ○ | 0 | orders at/above ship free (0 = never) |
| Zones | `zones[]` | rows | — | — | per **country / city** override; blank city = whole country |
| Show free-delivery banner | `show_free_delivery_banner` | toggle | ○ | true | storefront header banner promoting the free-delivery threshold |

**Storefront effect:** the checkout **country / city** selectors pick the matching zone; the header shows the free-delivery banner when enabled and a threshold is set.

### Pricing & discounts — settings section `pricing`

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Default markup % | default_markup_percent | number | ○ | 30 | — | — |
| Price rounding | price_rounding | select | ○ | "none" | none / nearest_0_05 / nearest_0_50 / nearest_1 / charm_99 | — |
| Max line discount % (cashier) | max_line_discount_percent | number | ○ | 10 | — | — |
| Max order discount % (cashier) | max_order_discount_percent | number | ○ | 15 | — | — |
| Require manager approval above limits | require_manager_override | toggle | ○ | true | — | — |
| Allow selling below cost | allow_below_cost | toggle | ○ | false | — | — |

### Purchasing — settings section `purchasing`

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Default supplier payment terms (days) | default_payment_terms_days | number | ○ | 30 | — | — |
| Require approval before sending a PO | require_po_approval | toggle | ○ | true | — | — |
| Approval required above amount | po_approval_threshold | number | ○ | 0 | 0 = every PO needs approval | — |
| Allow receiving more than ordered | allow_over_receipt | toggle | ○ | false | — | — |
| Update item cost on goods receipt | auto_update_cost_on_receipt | toggle | ○ | true | — | — |
| Costing method | costing_method | select | ○ | "weighted_average" | weighted_average / last_cost | — |

### Bank transfer — settings section `payments`

Bank-account details shown to customers who pick **bank transfer** at online checkout. Owner/admin only.

| UI label | Field key | Type | Req | Default | Notes |
|---|---|---|---|---|---|
| Bank name | bank_name | text | ○ | "" | — |
| Account title | account_title | text | ○ | "" | — |
| Account number | account_number | text | ○ | "" | — |
| IBAN | iban | text | ○ | "" | — |
| Transfer instructions | instructions | textarea | ○ | "" | Shown to customers who choose bank transfer at checkout |
| Payment reference note | reference_note | text | ○ | "" | Optional — e.g. "Use your order number as the reference." |

**Connections** — surfaced on the storefront checkout when the shopper selects *Bank transfer*, and on the order confirmation so they can complete the transfer.

### Expenses — settings section `expenses`

Approval policy for expenses recorded in Accounting. Owner/admin only.

| UI label | Field key | Type | Req | Default | Notes |
|---|---|---|---|---|---|
| Require approval before expenses post to the ledger | require_approval | toggle | ○ | false | When off, expenses post immediately (default) |
| Only require approval at or above this amount | approval_threshold | number | ○ | 0 | `0` = every expense needs approval when the toggle is on |

**Connections** — governs *Record expense* (`/admin/accounting`): above-threshold expenses wait for approval before posting to the ledger.

### Customers & credit — settings section `customers`

![Customers list](docs/screenshots/customers/customers-list.png)

Governs customer accounts and whether staff may sell on **credit / udhaar (pay later)** at the till. Owner/admin only.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Allow credit sales (udhaar) | credit_allowed | toggle | ○ | false | — | Master switch; when off the POS credit tender is hidden |
| Default credit limit (suggestion) | default_credit_limit | number | ○ | 0 | ≥ 0 | Pre-fills the limit when creating a customer; `0` = unlimited. Enforcement always uses each customer's own `credit_limit` |
| Who can sell on credit | credit_roles | multi-select (roles) | ○ | owner/admin/manager | role checklist | Only these roles see and can use the credit tender; owner/admin always included |

**Connections** — read by the POS *Payment screen* to decide whether the signed-in user may sell on credit; the unpaid balance of a credit sale posts to the customer's **Accounts Receivable** and is settled later from *Sales → Record payment*. See *Payment screen → Credit (pay later)*.

### Document numbering — settings section `documents`

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Sales order prefix | order_prefix | text | ○ | "SO-" | — | — |
| Receipt prefix | receipt_prefix | text | ○ | "RCP-" | — | — |
| Purchase order prefix | purchase_order_prefix | text | ○ | "PO-" | — | — |
| Stock transfer prefix | transfer_prefix | text | ○ | "TR-" | — | — |
| Number padding | number_padding | number | ○ | 5 | 5 → SO-00001 | — |
| Reset sequences each year | reset_yearly | toggle | ○ | false | — | — |

> This section writes only to `shops.settings.documents`. The live `document_sequences` rows (their own `prefix`/`padding`/`next_number`) are created at onboarding and are **not** edited here.

### Business hours — settings section `hours`
Weekday/weekend open & close times (text, e.g. "09:00"), closed days (comma-separated), holiday notice (textarea).

### Notifications & alerts — settings section `notifications`
Low stock alerts, Daily briefing, End-of-day summary — all toggles, default on.

### Branches & registers — read-only summary
Read-only overview of `branches` and `registers`; "Manage registers →" links to `/admin/registers`.

**Connections (Shop Settings overall)**
- **Selects / references:** POS section → *Branches* (`default_branch_id`) and *Registers* (`default_register_id`); Inventory → *Inventory locations* (`default_location_id`); Tax → *Tax rates*.
- **Feeds / used by:** `shops.settings` drives POS guardrails, pricing/discount limits, purchasing approvals, receipt rendering, storefront hours, and notifications. Business/localization columns render on receipts, invoices, storefront and admin/POS formatting.
- **Cross-form effects:** Business writes `shops` columns + `settings.business.tax_id`; other sections merge into `shops.settings[<section>]`; Tax-rate edits write `tax_rates` directly. Writes bypass the owner/admin-only `shops` RLS via the service-role client *after* the section role check (so managers can save their permitted sections).

**Tables owned:** `shops` (PK id; `created_by`→auth user; unique `slug`), `profiles` (PK id→auth.users), `shop_users` (`shop_id`→shops, `user_id`→auth user, `branch_id`→branches), `staff_invitations` (`shop_id`→shops, `branch_id`→branches, `invited_by`/`accepted_by`→auth users, unique `token`), `branches` (`shop_id`→shops), `document_sequences` (`shop_id`→shops, unique `(shop_id,doc_type)`), `storefront_settings` (`shop_id`→shops, unique). Provisioned/referenced: `registers`, `inventory_locations`, `tax_rates`, `payment_methods`, `accounts`, `ai_permissions`, `storefront_navigation`, `storefront_policies`, `expense_categories`, `audit_logs` (all `shop_id`→shops).

---

### Store Data Manager — `/admin/data-manager`

![Data manager](docs/screenshots/admin/data-manager.png)

- **Purpose:** Per-store **bulk data tools** — filter, bulk-update, bulk-delete, insert/seed and (owner-only) reset any of your store's own tables from one screen, without writing raw SQL.
- **Access:** **owner / admin** only; **Reset** is **owner-only**. Every action is **audited**. No schema change / migration — it operates on the existing tenant tables.
- **Backend:** server functions scoped to the **active shop**; writes can never reach another shop's rows.

**Workflow**
1. **Pick a table** — grouped as *Catalog*, *Customers & suppliers*, *Operations*, *Sales & finance*, *Config*. A couple (team members, audit log) are **view-only** for safety.
2. **Filter** — build conditions (**column · operator · value**); operators include equals / not-equals, greater / less, **contains**, **in (comma list)** and is-empty / not-empty, with a **Match ALL (AND) / Match ANY (OR)** toggle. A **compiled query preview** shows exactly what will run.
3. **Preview** — returns the matching **row count** plus a sample of rows before you change anything.
4. **Bulk update** sets one or more columns on every matching row; **Bulk delete** requires typing `DELETE` to confirm. Both **re-check the previewed count** before applying, are **capped at 50,000 rows**, and are audited.
5. **Add / seed rows** — paste **JSON** (an array of objects) or **CSV** (first row = column names); `shop_id` is added automatically.

**Foreign-key dependency resolver** — if a delete is blocked by related records (a foreign key), a popup lists the referencing rows per table with **Delete these** or **Set null (unlink)**. If clearing one of those is itself blocked, a **nested popup** opens — resolve down the chain, then delete again.

**Reset (owner only)** — wipe **Transactional**, **Catalog & CRM**, or **Everything** for the store, keeping setup/config (accounts, tax, registers, branches, locations, payment methods, storefront, team). Type `RESET` to confirm; a blocked reset offers the same dependency resolver.

**Safety model:** owner/admin only (Reset owner-only) · **own-store only** (always `shop_id`-scoped) · **no raw SQL** · typed confirmations (`DELETE` / `RESET`) · a 50,000-row cap · protected **read-only** tables (team members, audit log) · **every action audited**.

---

### Subscription & usage — `/admin/subscription`
- **Purpose:** Your free trial, per-module licenses, commercial plan and resource consumption in one place. **Backend:** `getShopResourceUsageFn` (`usage.functions.ts`) + the SaaS plan/billing functions.
- **Usage tab — Resource usage:** how much your store has consumed vs what remains, per category: **File storage** (product images, payment proofs, HR documents — enforced), **Database** (records across products, orders, customers, stock), **AI** (assistant messages, last 30 days), **Network** (data sent/received through the app in the last 30 days — uploads and storefront media; CDN-cached traffic isn't counted), **Team members** (enforced when your plan sets a limit), **Monthly active users** (staff + customer accounts that signed in during the last 30 days — informational, sign-ins are never blocked) and **Customers**.
- **Limits:** Your plan's numbers are real for three things — **file storage** (uploads stop with a clear "Storage limit reached" message at the cap, and a server-side storage policy is the hard backstop so the cap can't be bypassed), **team members** (inviting, accepting an invite or reactivating someone over the plan's user limit is refused with "You've reached your plan limit…") and **AI knowledge documents**. The remaining plan quotas in the table below stay informational. Any active license lifts the free-trial caps; the platform team can set individual allowances above your plan — contact support.
- **Need more?** Limits are adjustable per store: the platform team can set individual overrides (e.g. a larger storage allowance as part of your agreement) from the operator console — contact support. The Usage tab always shows the limits that actually apply to your store.
- **Module licenses & access:** Access to each of the 14 modules is governed by the **Module licenses** panel (managed by the platform/operator team, not editable by the shop). On a new store an active **free trial covers every module**; afterwards each module needs an active **license** (recorded when payment is received out-of-band). Month grants **stack** from the current period end; a **custom end date cannot shorten** an active license (it must be revoked first); a **Lifetime** grant never expires and absorbs later grants. Hard enforcement follows the platform's `LICENSING_ENFORCED` rollout flag — while it is off (the current default), licensing is informational and never blocks operational functionality. **Note:** holding any active module license moves the store from the **Trial** tier to the **Licensed** tier, which also **raises the enforced resource caps** (e.g. file storage from the trial 500 MB to the licensed allowance).

## Catalog

![Products list](docs/screenshots/catalog/products-list.png)

> **Available for sale (hide internal items).** Products, categories and brands each carry an **Available for sale** toggle (`is_online`, default **on**). Turn it **off** to make an item **internal-only** — e.g. raw materials, packing supplies, or a supplier-only brand/category: it is hidden from **both** the online storefront **and** POS product search, while staying fully usable in inventory, purchasing and the admin catalog. This is separate from **Active** (the archive flag): *Active off* = archived everywhere (also removed from POS and product selectors); *Available for sale off* = kept internally but not sold. Because the default is **on**, existing items are unaffected until you switch it off.

### New / Edit Product — `/admin/products`
- **Purpose:** Create or edit a catalog product with its media, merchandising flags, tax default and full variant matrix.
- **Access:** owner, admin, manager (server `requireRole(..., MANAGERS)`). Page requires auth; save is role-gated.
- **Backend:** `saveProductFn` (`src/lib/catalog.functions.ts`) → `saveProduct` (`src/lib/catalog.server.ts`) → `products` (+ cascades to `product_variants`, `variant_costs`, `prices`, `barcodes`, `inventory_movements`, `inventory_balances`). Images via `saveProductImagesFn` → `product_images`.

**Fields (product level)**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| — | `shopId` | uuid | Yes | active shop | — | `auto` |
| — | `productId` | uuid | No | null | — | `auto`; edit only |
| Product name | `name` | text | Yes | — | 1–200 chars | drives `slug` + SKU seed |
| Category | `category_id` | uuid (select) | No | null | EntitySelect `categories` | FK → categories |
| Brand | `brand_id` | uuid (select) | No | null | EntitySelect `brands` | FK → brands |
| Selling unit | `unit` | text | No | `pcs` | max 24 | server falls back to `pcs` |
| Status | `status` | enum | Yes | `active` (UI) | active / draft | DB column default `draft` |
| Short description | `short_description` | text | No | null | max 500 | — |
| Full description | `description` | textarea | No | null | max 20000 | — |
| Tags | `tags` | string[] | No | `[]` | ≤30 tags, each 1–40 | comma-separated input |
| Tax rate | `tax_rate_id` | uuid (select) | No | null | EntitySelect `tax_rates` | resolves variant→product→category→shop default |
| Available for sale | `is_online` | boolean | No | true | — | OFF = internal-only — hidden from storefront **and** POS search; still usable in inventory/purchasing. Separate from Status (archive) |
| Featured | `is_featured` | boolean | No | false | — | merchandising flag |
| New arrival | `is_new_arrival` | boolean | No | false | — | — |
| Best seller | `is_best_seller` | boolean | No | false | — | — |
| — | `slug` | text | — | slugify(name) | unique per shop | `auto` on create only |
| — | `has_variants` | boolean | — | variants.length>1 | — | `auto` computed |
| Media | (images) | sub-form | No | — | ≤30 images | see Product Images |
| Variants | `variants` | sub-form | Yes | — | min 1 | see Variant Builder |

**Variant Builder sub-form** (`src/components/admin/VariantBuilder.tsx`)

| UI label | Field key | Type | Req | Default | Validation / options | Notes / target |
|---|---|---|---|---|---|---|
| — | `id` | uuid | No | null | — | `auto`; edit only |
| Variant | `name` | text | No | null | max 120 | `auto` = attribute combo label ("Black / M") |
| SKU | `sku` | text | Yes | `<BASE>-<suffix>` | 1–80; unique per shop; no in-product dupes | `auto`-seeded → `product_variants.sku` |
| Barcode | `barcode` | text | No | null | max 64; unique per shop | → `barcodes` (is_primary); "Gen" button generates |
| Cost | `cost` | number | No | 0 | 0–10,000,000 | → `variant_costs.cost_price` & `avg_cost` (never public) |
| Price | `price` | number | No | 0 | 0–10,000,000 | → `product_variants.price` |
| Sale price | `sale_price` | number | No | null | 0–10,000,000 | >0 → `prices` (type='sale'); ≤0/empty deactivates |
| Weight | `weight` | number | No | null | 0–1,000,000 | → `product_variants.weight` |
| Tax | `tax_rate_id` | uuid (select) | No | null | tax rate list | collected + bulk-appliable but **not persisted per-variant** |
| Opening stock | `opening_stock` | number | No | 0 | 0–10,000,000 | create-only → `inventory_movements` (opening) + `inventory_balances` + a balanced GL journal (Dr Inventory 1200 / Cr Owner Equity 3000) for its value |
| Active | `is_active` | boolean | Yes | true | — | → `product_variants.is_active` |
| — | `attributes` | record<string,string> | — | `{}` | from attribute matrix | → `product_variants.attributes` (jsonb) |
| — | `is_default` | boolean | — | index===0 | — | `auto`; first variant is default |

Attribute matrix: the attribute-name field is a **creatable combobox** — choose an existing attribute (EntitySelect `attributes`), or **type a new name and press Enter** to create it inline (`ensureAttributeFn` → `product_attributes` + `product_attribute_values`); then tick its values → **Generate variants** builds the cartesian product. A bulk-apply bar sets price/cost/sale_price/weight/opening_stock/tax_rate_id/is_active across all or selected rows.

> **Number fields select-on-focus.** Numeric inputs in the builder (cost, price, sale price, weight, opening stock) select their existing value when you click into them, so typing cleanly replaces a leading "0".

> **Opening stock posts to the ledger.** Entering opening stock with a cost sets on-hand quantity and average cost **and** books a balanced journal entry **Dr Inventory (1200) / Cr Owner Equity (3000)** for the opening value (Σ opening_stock × cost) when the product is created, so the GL inventory asset matches the operational balance. Posting is best-effort — if the shop's chart of accounts lacks those accounts or the target period is closed, the stock still saves and the journal is skipped. (Changing a product's stock *later* via the editor posts a quantity **adjustment** only, with no journal — matching manual stock adjustments; value inventory changes through a **Purchase order → Supplier invoice** or an opening-balance journal.)

**Product Images sub-form** (`saveProductImagesFn`)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (upload) | `url` | text | Yes | — | 1–1000 chars | public URL (bucket `product-images`) |
| — | `storage_path` | text | No | null | max 400 | storage key |
| Alt | `alt` | text | No | null | max 200 | — |
| — | `id` | uuid | No | null | — | `auto` on existing rows |
| — | `position` | int | — | array index | — | `auto`; index 0 = cover |

**Connections**
- **Selects / references:** Category (`category_id`), Brand (`brand_id`), Tax rate (`tax_rate_id`), Attributes (`product_attributes`/`product_attribute_values`).
- **Feeds / used by:** POS (barcode lookup, sale lines; internal-only `is_online=false` items are excluded from POS product search), Storefront (status=active **and** available-for-sale), Inventory (balances/movements), Purchasing, Labels.
- **Cross-form effects:** opening stock → `inventory_movements`(opening)+`inventory_balances`+`journal_entries` (Dr Inventory 1200 / Cr Owner Equity 3000); cost → `variant_costs` (private); price → `product_variants.price` (fires `pricing_version` trigger); sale price → `prices`; barcode → `barcodes`.

> **Save errors show the real reason.** When a save or submit fails — here or anywhere else in the admin — the message now names the actual cause (e.g. a duplicate SKU, a validation problem, or a permission block) instead of a generic "Could not save…".

---

### Barcode generate / attach — `/admin/products` (variant panel + builder "Gen")
- **Purpose:** Generate or manually attach EAN-13 / CODE128 barcodes to a variant; deactivate old ones.
- **Access:** panel (generate/attach/deactivate) = owner, admin, manager, inventory; builder "Gen" & bulk-generate = owner, admin, manager.
- **Backend:** panel → `generateBarcodeFn` / `attachBarcodeFn` / `deactivateBarcodeFn` / `listVariantBarcodesFn` (`src/lib/barcode.functions.ts`); builder/bulk → `generateBarcodeFn` / `bulkGenerateBarcodesFn` (`src/lib/catalog.functions.ts`). Both → `barcodes`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| — | `shopId` | uuid | Yes | active shop | — | `auto` |
| — | `variantId` | uuid | Yes | — | — | target variant |
| (Generate EAN-13 / CODE128) | `type` | enum | Yes | `ean13` | ean13 / code128 | — |
| Attach manual barcode | `code` | text | Yes (attach) | — | 1–64; EAN-13 needs valid GS1 check digit; CODE128 ≤48 | duplicate active code rejected |
| — | (check digit) | — | auto | — | GS1 mod-10 (`ean13CheckDigit`) | `auto` |
| — | `source` | text | auto | generated / manual | — | `auto` per path |
| — | `is_primary` | boolean | auto | true if first active | — | deactivate promotes next barcode |
| — | `is_active` | boolean | auto | true | — | soft-delete keeps history |

Generation uniqueness: `barcode.server` uses a shop-scoped sequence (`ean13FromSequence`, prefix `20`); `catalog.server` uses random `makeEan13` (prefix `200`). "Bulk-generate missing" issues EAN-13 for every variant lacking one. **Scan to find** (Products page): `lookupBarcodeFn` resolves an active code to variant+product for search/POS.

---

### Category create / edit — `/admin/categories`
- **Purpose:** Manage hierarchical catalog categories (menu, category pages, product grouping).
- **Access:** UI `permission="products"`; server write INVENTORY_ROLES.
- **Backend:** `saveMasterDataFn` (entity `categories`) → `categories`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Name | `name` | text | Yes | — | required | — |
| Parent category | `parent_id` | uuid (entity) | No | null | EntitySelect `categories`; not self | FK → categories |
| Slug | `slug` | text | No | slugify(name) | unique per shop | `auto` when blank; dedup-suffixed |
| Display order | `position` | number | No | 0 | int | sort order |
| Default tax rate | `tax_rate_id` | uuid (entity) | No | null | EntitySelect `tax_rates` | category-level tax fallback |
| Category image | `image_url` | image | No | null | upload folder `categories` | — |
| Description | `description` | textarea | No | null | — | — |
| Available for sale | `is_online` | boolean | No | true | — | OFF = internal-only category — hidden from storefront **and** POS; still usable in inventory/purchasing |
| Visible on storefront | `is_active` | boolean | No | true | — | archive toggle |

**Connections** — self `parent_id`; `tax_rate_id`→tax_rates; used by products (`category_id`), storefront menu/category pages, filters. Delete blocked while referenced by products or child categories (archive instead).

---

### Brand create / edit — `/admin/brands`
- **Access:** UI `permission="products"`; server write INVENTORY_ROLES. **Backend:** `saveMasterDataFn` (entity `brands`) → `brands`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Name | `name` | text | Yes | — | required | — |
| Slug | `slug` | text | No | slugify(name) | unique per shop | `auto` when blank |
| Brand logo | `logo_url` | image | No | null | folder `brands` | — |
| Description | `description` | textarea | No | null | — | — |
| Available for sale | `is_online` | boolean | No | true | — | OFF = internal-only brand — hidden from storefront **and** POS; still usable in inventory/purchasing |
| Visible on storefront | `is_active` | boolean | No | true | — | archive toggle |

**Connections** — used by products (`brand_id`), storefront brand filters/pages. Delete blocked while products reference it.

---

### Attribute + values — `/admin/attributes`
- **Purpose:** Define variant attributes (Size, Colour…) and allowed values; feed the variant matrix.
- **Access:** UI `permission="products"`; server write INVENTORY_ROLES.
- **Backend:** attributes → `saveMasterDataFn` (entity `attributes`) → `product_attributes`; values → `saveMasterDataFn` (entity `attribute_values`) → `product_attribute_values`. The variant builder's attribute-name field is a creatable combobox that adds new attributes inline via `ensureAttributeFn`.

**Attribute**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Attribute name | `name` | text | Yes | — | required | — |
| Code | `code` | text | No | slugify(name) | unique per shop | `auto` when blank |

**Attribute value sub-form**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Value | `value` | text | Yes | — | required | — |
| — | `attribute_id` | uuid | Yes | selected attribute | — | `auto` from panel scope |

**Connections** — value `attribute_id`→product_attributes; feeds VariantBuilder → `product_variants.attributes`. Attribute delete blocked while it has values.

---

### Tax rate create / edit — `/admin/tax`
- **Access:** UI `permission="settings"` (owner, admin); **server write MANAGERS** (manager can write via API though UI hides it).
- **Backend:** `saveMasterDataFn` (entity `tax_rates`) → `tax_rates`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Name | `name` | text | Yes | — | required | placeholder "VAT 15%" |
| Rate (%) | `rate` | number | Yes | 0 | numeric(6,3); step 0.001 | — |
| Prices include this tax | `is_inclusive` | boolean | No | false | — | — |
| Use as shop default | `is_default` | boolean | No | false | exclusive | `auto` unsets other defaults |
| Active | `is_active` | boolean | No | true | — | archive toggle |

**Connections** — used by products, categories, and variant/line tax resolution across POS & online; fires `tax_version` trigger. Delete blocked while referenced by products, categories or sold order lines; default must be reassigned before delete.

---

### Barcode labels print — `/admin/labels`
- **Purpose:** Select variants, pick a size/fields, print a barcode label sheet (no DB writes).
- **Access:** any authenticated shop member (reads only).
- **Backend:** `labelCatalogFn` (`src/lib/catalog.functions.ts`) reads `product_variants` (+ `products`, `barcodes`); render via `BarcodeLabel` (JsBarcode).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (variant select) | `selected` | checkbox set | — | none | per variant | drives label queue |
| Copies | `copies[id]` | number | No | 1 | min 1 | per selected variant |
| Label size | `sizeId` | enum | Yes | `40x30` | 40x30, 50x25, 58roll, a4-3x8, a4-4x10 | mm dims from `LABEL_SIZES` |
| Show → shop name/logo | `fields.shop` | boolean | No | true | — | — |
| Show → product name | `fields.productName` | boolean | No | true | — | — |
| Show → variant | `fields.variantName` | boolean | No | true | — | — |
| Show → SKU | `fields.sku` | boolean | No | true | — | — |
| Show → price | `fields.price` | boolean | No | true | — | — |
| Show → barcode | `fields.barcode` | boolean | No | true | — | format auto-resolved (EAN13/CODE128) |

**Connections** — reads primary `barcodes.code`, variant `price`/`sku`, product `name`, shop name/logo; output goes to `window.print` only.

---

### Bulk import — `/admin/import`
- **Purpose:** Import products / suppliers / inventory from CSV or XLSX: upload → map columns → validate → commit.
- **Access:** owner, admin, manager (validate & commit).
- **Backend:** `validateImportFn` / `commitImportFn` (`src/lib/import.functions.ts`). Products commit → `products`, `product_variants`, `variant_costs`, `prices`, `barcodes`, `inventory_movements`, `inventory_balances`.
- **Published on commit:** new products are created **live** — `status = active` and `is_online = true` — so they show on the storefront, restaurant menu and POS immediately (like manually-created products). Re-importing an existing product (matched by slug/SKU) updates it and leaves its current published state untouched.

**Wizard controls**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Type | `kind` | enum | Yes | `products` | products / suppliers / inventory | step 1 |
| File | (upload) | file | Yes | — | `.csv`, `.xlsx`, `.xls`; ≤5000 rows | parsed client-side |
| Column mapping | `mapping` | record | — | auto-guessed | file header → target field | per-field select |

**Products import target fields**

| UI label | Field key | Req | Validation / options | Notes / target |
|---|---|---|---|---|
| Product name | `product_name` | Yes | non-empty | groups rows into one product; slug-matched for upsert |
| SKU | `sku` | Yes | non-empty; dedup in file; existing → update | → `product_variants.sku` |
| Variant label | `variant` | No | — | → `product_variants.name` |
| Attributes | `attributes` | No | `Color:Red;Size:M` | parsed → jsonb attributes |
| Category | `category` | No | matched by name (warns if not found) | → `category_id` |
| Brand | `brand` | No | matched by name (warns if not found) | → `brand_id` |
| Cost | `cost` | No | numeric, default 0 | → `variant_costs` |
| Price | `price` | Yes | numeric | → `product_variants.price` |
| Sale price | `sale_price` | No | numeric | → `prices` (sale) |
| Opening stock | `opening_stock` | No | numeric, default 0 | new variants only → movements+balances + one GL journal (Dr Inventory 1200 / Cr Owner Equity 3000) for the imported opening value |
| Barcode | `barcode` | No | dedup in file; cross-product clash = error | → `barcodes` |

(Suppliers import → `suppliers`; Inventory import → `inventory_*`.) **Validate** cross-checks existing categories, brands, variant SKUs and barcodes, returning per-row ok/warning/error counts. **Commit** skips error rows, imports ok+warning, audit-logged; products grouped by name → one product with N variants.

**Tables owned:** `products` (`category_id`→categories, `brand_id`→brands, `tax_rate_id`→tax_rates; unique shop_id+slug), `product_variants` (`product_id`→products; unique shop_id+sku; extra DB cols not in UI: `compare_at_price`, `track_inventory`, `low_stock_threshold`, `reorder_point`, `reorder_quantity`), `product_attributes` (unique shop_id+code), `product_attribute_values` (`attribute_id`→product_attributes), `product_images` (`product_id`→products, `variant_id`→product_variants), `prices` (`variant_id`→product_variants; `price_type`='sale'), `variant_costs` (`variant_id`→product_variants; private cost), `barcodes` (`variant_id`→product_variants; unique shop_id+code, partial-unique on active; `type`/`source`/`is_primary`/`is_active`), `categories` (self-FK `parent_id`, `tax_rate_id`; unique shop_id+slug), `brands` (unique shop_id+slug), `tax_rates` (`is_default` exclusive). Import also touches `inventory_movements`/`inventory_balances` and `suppliers`.

---

## Inventory

![Stock movements](docs/screenshots/inventory/stock-movements.png)

### Manual stock adjustment — `/admin/inventory`
- **Purpose:** Post a signed (+/-) quantity correction against a variant at the shop's default location, with a reason.
- **Access:** Route is admin-authenticated; the write is a client-side Supabase insert/update governed by `inventory_movements`/`inventory_balances` RLS (inventory roles). See role matrix.
- **Backend:** inline `applyAdjustment` in the route → direct writes to `inventory_movements` + `inventory_balances`.

**Fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Product | `variant_id` | select (uuid) | Yes | "" | variants that already have a balance row | → movement + balance `variant_id` |
| Quantity (+/-) | `quantity` | text→number | Yes | "" | `Number(quantity)` must be non-zero; signed | positive adds, negative removes |
| Reason | `note` | text | No | "" | — | → `inventory_movements.note` |
| Location | `location_id` | — | auto | shop's `is_default` location | limit 1 | "No inventory location configured" if none |
| Movement type | `movement_type` | — | auto | `'adjustment'` | enum | fixed |
| On-hand after | balance `quantity` | — | auto | existing + delta | running balance | upserts `inventory_balances` |

**Connections**
- **Selects / references:** Variant via `variant_id`; Location implicit via default `location_id`.
- **Feeds / used by:** updated balance read by POS / Products / Dashboard low-stock; movement shown in "Recent movements".
- **Cross-form effects:** each post writes one `inventory_movements` row and updates/creates the `inventory_balances` row.

---

### Reorder / stock defaults — `/admin/settings` (Inventory section)
- **Purpose:** Shop-wide stock defaults (low-stock threshold, reorder point/quantity, default location, negative-stock rule).
- **Access:** see role matrix (settings-section gate).
- **Backend:** `updateShopSection` (`src/lib/shop.functions.ts`) → `shops.settings.inventory`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Allow negative stock | `allow_negative_stock` | toggle | No | false | — | lets sales pass below zero (and governs the online-store checkout) |
| Who can oversell (negative stock) | `oversell_roles` | multi-select (roles) | No | owner/admin/manager | role checklist | roles allowed to push a **till** sale below zero; others are capped at on-hand (owner/admin always included) |
| Default location | `default_location_id` | select | No | default location | active `inventory_locations` | "No default" allowed |
| Low-stock threshold (default) | `low_stock_threshold` | number | No | 5 | ≥ 0 | fallback when variant has none |
| Reorder point | `reorder_point` | number | No | 10 | ≥ 0 | used by AI reorder insights |
| Reorder quantity | `reorder_quantity` | number | No | 20 | ≥ 0 | suggested order size |

Per-variant overrides (`low_stock_threshold`, `reorder_point`, `reorder_quantity`) exist on `product_variants` but are **not** editable in the current UI. These feed AI reorder recommendations and Dashboard "Low stock priority" (variant value ?? default).

---

### Create transfer (draft → send → receive) — `/admin/transfers`
- **Purpose:** Move stock between two locations through draft → requested → approved → shipped → received.
- **Access:** create/edit/ship/receive = INVENTORY_ROLES (owner, admin, manager, inventory); approve = MANAGERS. Cancel disabled once shipped.
- **Backend:** `createTransferFn`, `updateTransferItemsFn`, `transitionTransferFn` (`src/lib/inventory-ops.functions.ts`) → `inventory_transfers`, `inventory_transfer_items`; on ship/receive → `postInventory` (`inventory_movements` + `inventory_balances`).

**Header fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| From location | `from_location_id` | select | Yes | "" | active locations; must differ from To | — |
| To location | `to_location_id` | select | Yes | "" | active locations, excludes From | — |
| Notes | `notes` | text | No | — | max 500 | — |

**Line items** (`transferItemSchema`)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Product | `variant_id` | select | Yes | "" | active variants (limit 500) | — |
| Qty | `quantity` | number | Yes | "" | 0.001–999999; rows ≤0 dropped; ≥1 line required | — |
| Avail | (computed) | — | auto | — | `balance.quantity − reserved` at From | display-only guide |
| Unit cost | `unit_cost` | number | No | 0 | 0–999999 | in schema/server; not in create UI |

**Auto/computed record fields:** `transfer_number` (`nextDocNumber(..., "transfer", "TR-")`, unique per shop), `status` (default `draft`; enum draft/requested/approved/shipped/received/cancelled), `created_by`, per-transition `*_by`/`*_at` stamps, and per-line `received_quantity` (default 0, set at receipt, editable). The Receive step sends `receivedQuantities` keyed by transfer-item id; Ship validates source availability.

**Connections**
- **Selects / references:** From/To via `from_location_id`/`to_location_id`; lines via `variant_id`.
- **Feeds / used by:** Ship posts `transfer_out` (−) at source; Receive posts `transfer_in` (+) at destination — both write `inventory_movements` + `inventory_balances`; audit `transfer.<status>`.
- **Cross-form effects:** stock only leaves source on **ship** and only lands at destination on **receive**; draft/requested/approved move no stock.

---

### Stock count (start → count → review → post) — `/admin/stock-counts`
- **Purpose:** Physical count vs expected, then post variance adjustments to balances.
- **Access:** create/save = INVENTORY_ROLES; approve = MANAGERS.
- **Backend:** `createStockCountFn`, `saveStockCountProgressFn`, `approveStockCountFn` (`src/lib/inventory-ops.functions.ts`) → `stock_counts`, `stock_count_items`; approve → `postInventory`.

**Create fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Location | `location_id` | select | Yes | "" | active locations | sets expected snapshot source |
| Scope | `mode` (UI only) | select | Yes | "all" | all / category | maps to fields below |
| — (all) | `all_products` | boolean | — | true when scope=all | — | seeds every variant |
| Category checkboxes | `category_ids` | uuid[] | cond. | [] | shown when scope=category | seeds variants of those categories |
| Note | `note` | text | No | — | max 500 | — |
| — | `variant_ids` | uuid[] | No | — | in schema, not in UI | explicit variant seed |

**Count-sheet line items** (`stock_count_items`)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Product | `variant_id` | — | auto | seeded at create | — | — |
| Expected | `expected_qty` | numeric(14,3) | auto | balance at creation | — | snapshot, not editable |
| Counted | `counted_qty` | number | Yes | 0 | ≥ 0 | editable until completed |
| Difference | (computed) | — | auto | counted − expected | — | highlighted if ≠ 0 |

**Save progress** sets `status='counting'`, `counted_by`. **Approve/post** posts a `count`-type movement for every non-zero diff (qty = counted − expected), sets `status='completed'`, `approved_by`, `approved_at`, `completed_at`. It then values the net variance at each variant's cost (`variant_costs.avg_cost`, else `cost_price`) and posts **one balanced GL journal** so the perpetual Inventory account tracks the reconciliation: shrinkage (stock down) posts **Dr COGS 5000 / Cr Inventory 1200**, overage (stock up) posts **Dr Inventory 1200 / Cr COGS 5000**. Period-guarded and best-effort — balances are the source of truth and are never rolled back if the chart of accounts lacks 1200/5000 or the period is closed.

**Connections**
- **Selects / references:** Location via `location_id`; items via `variant_id`; scope via categories.
- **Feeds / used by:** approved variances update `inventory_balances`; a balanced `journal_entries`/`journal_entry_lines` pair (`reference_type='stock_count'`) values the net variance to Inventory 1200 vs COGS 5000; audit `stock_count.approved`.
- **Cross-form effects:** posting a count writes `inventory_movements` (type `count`) + updates `inventory_balances` for each non-zero variance, and mirrors the net cost change to the ledger so the balance sheet Inventory line and P&L stay correct.

---

### Location create / edit — `/admin/locations`
- **Purpose:** Manage stock locations that hold balances and receive transfers/counts.
- **Access:** UI permission `inventory`; server write = INVENTORY_ROLES.
- **Backend:** `saveMasterDataFn` / `setMasterDataActiveFn` / `deleteMasterDataFn` (`src/lib/masterdata.functions.ts`, entity `inventory_locations`) → `inventory_locations`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Location name | `name` | text | Yes | — | placeholder "Main store" | — |
| Branch | `branch_id` | entity select | No | null | references `branches` | FK, nullable |
| Default location for sales | `is_default` | boolean | No | false | — | — |
| Active | `is_active` | boolean | No | true | — | archivable |

**Connections** — references Branch via `branch_id`; owns the `inventory_balances`/`inventory_movements` rows; supplies location options to transfers, counts, adjustments, and the settings default location.

---

### Branch create / edit — `/admin/branches`
- **Purpose:** Physical trading locations scoping staff, registers, stock locations and reporting.
- **Access:** UI permission `settings`; server write = ADMINS (owner, admin).
- **Backend:** `saveMasterDataFn` / `setMasterDataActiveFn` / `deleteMasterDataFn` (entity `branches`) → `branches`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Branch name | `name` | text | Yes | — | — | — |
| Code | `code` | text | No | null | — | — |
| Phone | `phone` | text | No | null | — | — |
| Address | `address` | textarea | No | null | — | — |
| Default branch | `is_default` | boolean | No | false | — | — |
| Active | `is_active` | boolean | No | true | — | — |
| — | `business_hours` | jsonb | — | `[]` | — | not in this UI |

**Connections** — referenced by `inventory_locations.branch_id` and `registers.branch_id`; drives branch scoping for locations, registers, staff and reporting.

**Tables owned:** `inventory_balances` (unique `(location_id, variant_id)`; FKs `location_id`→inventory_locations, `variant_id`→product_variants), `inventory_movements` (FKs `location_id`, `variant_id`; enum `movement_type`; polymorphic `reference_type`/`reference_id`), `inventory_locations` (FKs `shop_id`, `branch_id`→branches), `inventory_transfers` (unique `(shop_id, transfer_number)`; `from_location_id`/`to_location_id`→inventory_locations; enum `transfer_status`), `inventory_transfer_items` (`transfer_id`→inventory_transfers, `variant_id`→product_variants), `stock_counts` (FKs `shop_id`, `location_id`; `counted_by`/`approved_by`), `stock_count_items` (`stock_count_id`→stock_counts, `variant_id`→product_variants), `branches` (`shop_id`→shops), `registers` (`branch_id`→branches). Reorder defaults live in `shops.settings.inventory`, with nullable per-variant overrides on `product_variants`.

---

## Purchasing & Accounting

![Purchase orders](docs/screenshots/purchasing/purchase-orders.png)

Two admin pages hold most of these forms: **Purchasing & suppliers** (`/admin/purchasing`) and **Accounting** (`/admin/accounting`). The master-data pages (Suppliers, Chart of accounts, Expense categories, Payment methods) share the generic `MasterDataPage` component → `saveMasterDataFn` / `setMasterDataActiveFn` / `deleteMasterDataFn` (`src/lib/masterdata.functions.ts`). Financial writes (invoices, payments, expenses) run through server functions gated by **FINANCE_ROLES** (owner, admin, manager, accountant).

### Supplier — quick add — `/admin/purchasing`
- **Purpose:** Fast supplier capture inline on the purchasing page.
- **Access:** purchasing roles (client insert governed by `suppliers` RLS).
- **Backend:** direct `supabase.from("suppliers").insert(...)`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Name | `name` | text | Yes | "" | non-empty | — |
| Email | `email` | text | No | "" | — | — |
| Phone | `phone` | text | No | "" | — | — |
| — | `payment_terms` | text | auto | `net_30` | — | set even though no input is shown here |

### Supplier — full record — `/admin/suppliers`
- **Purpose:** Maintain the supplier master used by POs, receipts and invoices.
- **Access:** UI `permission="purchasing"`. **Backend:** `saveMasterDataFn` (entity `suppliers`) → `suppliers`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Supplier name | `name` | text | Yes | — | required | — |
| Code | `code` | text | No | — | — | — |
| Email | `email` | text | No | — | — | — |
| Phone | `phone` | text | No | — | — | — |
| Tax number | `tax_number` | text | No | — | — | — |
| Payment terms | `payment_terms` | select | No | — | cod / net_7 / net_15 / net_30 / net_60 | — |
| Credit limit | `credit_limit` | number | No | — | step 0.01 | — |
| Address | `address` | textarea | No | — | — | — |
| Active | `is_active` | boolean | No | true | — | archive toggle |

**Connections** — referenced by `purchase_orders.supplier_id`, `supplier_invoices.supplier_id`; delete blocked while purchase orders reference it (`masterdata.server` guard).

---

### Purchase order — `/admin/purchasing` (New purchase order)
- **Purpose:** Raise a PO against a supplier with **one or more** product lines, each with its own quantity and unit cost.
- **Access:** purchasing roles (client writes governed by RLS).
- **Backend:** direct client writes → `purchase_orders` + `purchase_order_items`; PO number from `document_sequences` (doc_type `purchase_order`).

Pick a supplier, then add one **row per product**, each with its own quantity and unit cost. Use **+ Add product** to add rows and the row's remove control to delete them; a **live running total** sums every line as you type. The same product can't appear twice on one PO — combine it into a single line (duplicates are blocked).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Supplier | `supplier_id` | uuid (EntitySelect) | No | "" → null | from *Suppliers* | one supplier per PO |
| Product | `variant_id` | select | Yes | "" | active variants (limit 500) | one row per product; no duplicate products in a PO |
| Quantity | `quantity` | number | Yes | 1 | `Number()||1` | per line; input selects-on-focus |
| Unit cost | `unit_cost` | number | Yes | "" | `Number()||0` | per line; input selects-on-focus |
| — | `po_number` | text | auto | `PO-#####` | `document_sequences` prefix+padding, increments `next_number` | `auto`; preserved when editing |
| — | `status` | enum | auto | `submitted` | po_status | `auto`; preserved when editing |
| — | `subtotal` / `total` | number | auto | Σ(quantity × unit_cost) across lines | — | `auto`; the live running total |
| — (line) | `total` | number | auto | quantity × unit_cost | — | `auto` |

> The AI assistant can also create multi-line POs via `create_purchase_order` (`ai-actions.server.ts`) subject to the AI permission matrix.

**Expandable detail, Edit & Cancel.** In the purchase-orders list, **click a PO row** to expand an inline detail table with one row per line — **Product · Ordered · Received · Unit cost · Line total**. From that expanded detail, and **only while nothing has been received** on the PO, two actions are available:
- **Edit** — reopens the form pre-filled so you can change the supplier and replace the line items; the **PO number and status are preserved**.
- **Cancel PO** — sets the status to `cancelled` after a confirm prompt.

Once any goods are received, Edit and Cancel disappear and a short note explains the order can no longer be changed — editing a received PO would require reversing already-posted stock and cost postings, which isn't supported.

**Connections** — references Supplier (`supplier_id`) and Variant (`variant_id`, per line); consumed by *Receive goods* and *Supplier invoice* (prefill).

---

### Receive goods — `/admin/purchasing` (per-PO "Receive goods" button)
- **Purpose:** Post a goods receipt for an open PO — full ordered quantity, no form fields.
- **Access:** purchasing roles. **Backend:** direct writes → `goods_receipts` + `goods_receipt_items`, `inventory_movements` (type `purchase`), `inventory_balances`, `variant_costs` (upsert), PO `status='received'`; GRN number from `document_sequences` (doc_type `goods_receipt`).

| Field | Type | Req | Default | Notes |
|---|---|---|---|---|
| `purchase_order_id` | uuid | auto | the PO | from button |
| `location_id` | uuid | auto | shop default location | "No inventory location configured" if none |
| `receipt_number` | text | auto | `GRN-#####` | `document_sequences` |
| per line `variant_id`, `quantity`, `unit_cost` | — | auto | from PO items | received at full ordered qty |

**Cross-form effects** — each line adds stock (`inventory_movements` purchase + `inventory_balances`), updates cost (`variant_costs.cost_price`/`avg_cost`), stamps `purchase_order_items.received_quantity`, and flips the PO to `received`. A goods receipt updates stock and cost **operationally only** — it posts **no journal entry**; the inventory asset and the payable are booked when the matching **supplier invoice is posted** (invoice-time recognition).

---

### Supplier invoice — `/admin/purchasing` (New supplier invoice)
- **Purpose:** Record a supplier bill (draft), then Post / Cancel it.
- **Access:** FINANCE_ROLES. **Backend:** `saveSupplierInvoice` / `postSupplierInvoiceFn` / `cancelSupplierInvoiceFn` / `createInvoiceFromSourceFn` (`src/lib/purchasing.functions.ts`) → `supplier_invoices`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Supplier | `supplier_id` | uuid (EntitySelect) | Yes | "" | from *Suppliers* | — |
| Invoice number | `invoice_number` | text | Yes | "" | 1–80 chars | — |
| Invoice date | `invoice_date` | text (date) | No | — | optional | — |
| Due date | `due_date` | text (date) | No | — | nullable | drives AP ageing / due-soon |
| Total | `total` | number | Yes | "" | ≥ 0 | — |
| Prefill from purchase order | `purchase_order_id` | select | No | — | your POs | calls `createInvoiceFromSourceFn` to draft from a PO/receipt |
| — | `goods_receipt_id` | uuid | No | null | schema only | alternate source |
| — | `notes` | text | No | null | ≤2000 | schema only |
| — | `status` | text | auto | `draft` | draft → posted → partially_paid / paid / overdue / cancelled | `auto` |
| — | `paid_total` | number | auto | 0 | — | `auto`; updated by payments |

**Actions:** **Post** (draft → posted, writes the journal entry **Dr Inventory / Cr Accounts Payable** for stock items — this is when purchased stock becomes a GL asset), **Cancel** (only when nothing paid). **Connections** — Supplier (`supplier_id`), optional PO (`purchase_order_id`) / goods receipt (`goods_receipt_id`); feeds AP ageing and "Outstanding by supplier".

---

### Supplier payment — `/admin/purchasing` (Record a payment)
- **Access:** FINANCE_ROLES. **Backend:** `recordSupplierPaymentFn` → `supplier_payments` (+ journal, updates invoice `paid_total`/`status`).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Invoice | `supplier_invoice_id` | select | Yes | "" | invoices in posted / partially_paid / overdue | shows outstanding per option |
| Amount | `amount` | number | Yes | "" | > 0 | — |
| Method | `method_code` | select | Yes | `cash` | cash / bank / card | — |
| Reference | `reference` | text | No | "" | ≤200 | nullable |
| — | `paid_at` | text (date) | auto | now | optional | `auto` |

**Cross-form effects** — increments the invoice's `paid_total`, moves it to partially_paid / paid, and posts a balanced journal entry (Dr AP / Cr Cash or Bank).

---

### Record expense — `/admin/accounting` (Overview tab)
- **Purpose:** Record an operating expense and post it to the ledger.
- **Access:** FINANCE_ROLES. **Backend:** `recordExpense` (`src/lib/accounting.functions.ts`) → `recordExpenseAndPost` → `expenses` + `journal_entries`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Category | `category_id` | select | No | "" → null | from *Expense categories* | maps to a ledger account |
| Amount | `amount` | number | Yes | "" | > 0 | — |
| Note | `note` | text | No | "" | ≤500; ""→null | — |
| Payment method | `payment_method_code` | select | Yes | `cash` | cash / bank | default `cash` |
| — | `branch_id` | uuid | auto | top-bar Branch filter | nullable | taken from the page's branch filter |

**Cross-form effects** — writes an `expenses` row and a balanced journal entry (Dr the category's expense account / Cr Cash or Bank). Also reachable via a cash-drawer expense on `/admin/registers` (see *Cash movement*).

---

### Manual voucher (journal / payment / receipt / contra) — `/admin/accounting` (Vouchers tabs)
- **Purpose:** Post manual double-entry that the automated flows don't generate — owner capital, bank charges, depreciation, accruals, reclassifications, opening balances.
- **Access:** create / edit / view / delete = FINANCE_ROLES; **approve / post / reverse = MANAGERS** (owner/admin/manager). **Backend:** `createVoucher` / `updateVoucher` / `approveVoucher` / `postVoucher` / `reverseVoucher` (`src/lib/vouchers.functions.ts` → `vouchers.server.ts`) → `vouchers` + `voucher_lines`; posting writes `journal_entries` / `journal_entry_lines`.

**Header fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (tab) | `voucher_type` | — | Yes | `journal` | journal / payment / receipt / contra (+ sales / purchase / debit_note / credit_note) | picks the number series (JV-/PV-/RV-/CV-…) |
| Date | `voucher_date` | date | Yes | today | ISO; target period must be open | period-guarded on post |
| Reference | `reference` | text | No | — | ≤120 | e.g. cheque #, invoice ref |
| Narration | `narration` | text | No | — | ≤1000 | what the voucher is for |
| Notes | `notes` | text | No | — | ≤2000 | internal only |

**Line grid** (`voucher_lines`, **min 2 rows**)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Account | `account_id` | select | Yes | — | searchable Chart of Accounts | — |
| Debit | `debit` | number | — | 0 | ≥ 0 | a line is debit **or** credit, never both |
| Credit | `credit` | number | — | 0 | ≥ 0 | — |
| Department | `department_id` | select | No | null | active departments | analytical dimension |
| Cost center | `cost_center_id` | select | No | null | active cost centers | analytical dimension |
| Narration | `narration` | text | No | — | ≤300 | per-line memo |

**Balanced-entry enforcement** — the totals row shows **"Out by ₨X"** (amber) until debits = credits, then flips to **"Balanced"**; **Save is disabled while unbalanced**. `validateLines` re-checks on the server (and the `assert_journal_balanced` DB trigger on post): rejects <2 lines ("A voucher needs at least two lines."), a line with both debit and credit, and any imbalance ("Voucher is not balanced…"). **Type-specific rules:** *payment* = exactly one Cash/Bank line, credited; *receipt* = exactly one Cash/Bank line, debited; *contra* = exactly two Cash/Bank lines (different accounts), one debit + one credit.

**Lifecycle** — `draft → approved → posted → reversed`. *Save draft* creates the voucher with **no ledger impact**. Open it (View) → **Approve** (managers) → **Post** — only posting writes the journal entry and moves the trial balance. A posted voucher shows **Reverse**, which writes an equal-and-opposite (debits↔credits swapped) contra entry, period-guarded — never by editing the original. *Delete* is allowed only while draft.

**Cross-form effects** — posted vouchers appear in General Ledger / Trial Balance / P&L / Balance Sheet identically to automated postings; `AUD` `voucher.*` actions logged; department/cost-center dimensions feed the Departments and Cost Centers report tabs.

---

### Chart of accounts — `/admin/accounts`
- **Access:** UI `permission="accounting"`. **Backend:** `saveMasterDataFn` (entity `accounts`) → `accounts`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Account code | `code` | text | Yes | — | e.g. 1000; locked on system accounts | — |
| Account name | `name` | text | Yes | — | required | — |
| Type | `type` | select | Yes | — | asset / liability / equity / income / expense; locked on system | — |
| Parent account | `parent_id` | uuid (entity) | No | null | EntitySelect `accounts` | tree self-reference |
| Active | `is_active` | boolean | No | true | — | archive toggle |

**Connections** — `journal_entry_lines.account_id`, `expense_categories.account_id`, `cash_accounts.account_id`, self `parent_id`. System accounts (`is_system`) are protected from deletion; the default 14-line chart is provisioned at onboarding.

---

### Expense category — `/admin/expense-categories`
- **Access:** UI `permission="accounting"`. **Backend:** `saveMasterDataFn` (entity `expense_categories`) → `expense_categories`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Name | `name` | text | Yes | — | e.g. "Rent" | — |
| Ledger account | `account_id` | uuid (entity) | No | null | EntitySelect `accounts` | expense account this category posts to |
| Active | `is_active` | boolean | No | true | — | archive toggle |

**Connections** — selected on *Record expense* (`category_id`); each category maps to an `accounts` row via `account_id`.

---

### Payment method — `/admin/payments`
- **Access:** UI `permission="settings"`. **Backend:** `saveMasterDataFn` (entity `payment_methods`) → `payment_methods`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Display name | `name` | text | Yes | — | e.g. "Cash" | — |
| Code | `code` | text | Yes | — | stable id stored on payments (cash, card…) | — |
| Type | `type` | select | Yes | — | cash / card / bank / wallet / credit / other | cash types post to the drawer + cash ledger |
| Display order | `position` | number | No | — | — | — |
| Enabled | `is_enabled` | boolean | No | true | — | — |

**Connections** — codes are referenced by POS payments, supplier payments and expenses. (Note: storefront checkout uses a hardcoded cod/bank/card list, **not** this table.)

---

### Financial reports (read-only) — `/admin/accounting`

![Accounting ledger](docs/screenshots/accounting/reports-ledger.png)
Tabs computed from the double-entry ledger (`journal_entries` / `journal_entry_lines`), not from UI sales totals. Shared filters: **From / To** date range, **Branch**, and (Ledger tab) **Account**; a **Print current report** button.

| Tab | Server fn | Filters |
|---|---|---|
| Overview | P&L + Balance sheet + Trial balance | range, branch |
| P&L | `getProfitAndLoss` | range, branch |
| Balance Sheet | `getBalanceSheet` | as-of (range end), branch |
| Cash Flow | `getCashFlow` | range, branch |
| General Ledger | `getGeneralLedger` | range, account, limit 500 |
| Trial Balance | `getTrialBalance` | range, branch |
| AR Ageing | `getArAgeing` | as-of |
| AP Ageing | `getApAgeing` | as-of |

A ledger-integrity banner (`getBalanceValidation`) warns if total debits ≠ total credits in the range.

> The same `/admin/accounting` page also hosts operational tabs beyond these read-only reports: **Journal / Payment / Receipt / Contra Vouchers** (see *Manual voucher* above), **Fiscal Years / Periods** (open/close periods — the posting guard that blocks back-dating into a closed month), **Departments** and **Cost Centers** (analytical dimensions tagged on voucher lines), and **Bank Reconciliation** (match statement lines against ledger cash/bank movements until the difference is zero).

**Tables owned:** `suppliers` (`shop_id`→shops), `purchase_orders` (`shop_id`→shops, `supplier_id`→suppliers; `po_status`; unique `(shop_id, po_number)`), `purchase_order_items` (`purchase_order_id`→purchase_orders, `variant_id`→product_variants), `goods_receipts` (`purchase_order_id`→purchase_orders, `location_id`→inventory_locations; unique `(shop_id, receipt_number)`), `goods_receipt_items` (`goods_receipt_id`→goods_receipts, `variant_id`→product_variants), `supplier_invoices` (`supplier_id`→suppliers, `purchase_order_id`→purchase_orders, `goods_receipt_id`→goods_receipts; `payment_status`), `supplier_payments` (`supplier_invoice_id`→supplier_invoices), `accounts` (self-FK `parent_id`; `account_type`; `is_system`), `journal_entries` (`shop_id`→shops, `branch_id`→branches), `journal_entry_lines` (`journal_entry_id`→journal_entries, `account_id`→accounts), `expenses` (`category_id`→expense_categories, `branch_id`→branches), `expense_categories` (`account_id`→accounts), `payment_methods` (`shop_id`→shops), `bank_accounts` (`account_id`→accounts).

---

### Budgets — `/admin/budgets`
- **Purpose:** Annual budgets by account (optionally per month) with **Budget vs Actual** computed against the live ledger (posted actuals only).
- **Access:** view / edit draft lines = FINANCE_ROLES; **Approve = owner/admin**. **Backend:** `createBudget` / `getBudget` / `upsertBudgetLine` / `setBudgetStatus` / `budgetVsActual` (`src/lib/finance-reports.functions.ts` → `finance-reports.server.ts`) → `budgets` + `budget_lines`.

**Create + line fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Budget name | `name` | text | Yes | — | non-empty | e.g. "FY2026 Operating Budget"; created as `draft`, version 1 |
| Account code | `account_code` | text | Yes | — | must exist in Chart of Accounts (resolved to `account_id` server-side) | e.g. `6200` |
| Amount | `amount` | number | Yes | — | numeric **> 0** (letters/blank → "Enter a numeric amount greater than zero.") | per-account budgeted figure |

**Lines** — added on the **Budget lines** tab while the budget is `draft`. Enter an **account code + amount** and press **Add / update line**: the code is resolved against the Chart of Accounts server-side and the line is **upserted** (re-entering the same code updates its amount instead of duplicating). Lines become read-only once the budget is approved.

**Budget vs Actual** — the second tab compares each budgeted account to its **posted** ledger actual (year-to-date): Budget, Actual, Variance and Variance %, with over-budget variance highlighted amber. (Procurement requisitions can reference a budget for encumbrance context.)

**Lifecycle** — `draft → approved` (**Approve**, owner/admin). Only draft budgets accept line edits.

**Connections** — `budget_lines.account_id`→accounts; `budgets.fiscal_year_id`→fiscal_years (optional); actuals read from posted `journal_entry_lines`.

---

### Receivables & Payables (ageing statements) — `/admin/receivables`, `/admin/payables`
- **Purpose:** Read-only ageing of customer (AR) and supplier (AP) outstanding balances, bucketed **Current / 1–30 / 31–60 / 61–90 / 90+** as of a chosen date. Click a row to open that party's running **statement** (opening balance → dated debits/credits → closing balance).
- **Access:** FINANCE_ROLES. **Backend:** `getArAgeing` / `getApAgeing` + per-party statement (`finance-reports.server.ts`), derived from sales & their tenders (AR — **cash sales excluded**) and `supplier_invoices` / `supplier_payments` (AP).
- **Settlement is done elsewhere** — these pages are read-only. Clear an **AR** balance from *Sales → Record payment* on the sale; clear an **AP** balance from *Purchasing → Record a payment* on the supplier invoice. The balance drops to zero once the settling payment posts (**Dr Bank/Cash · Cr Accounts Receivable**, or **Dr Accounts Payable · Cr Bank/Cash**), and the trial balance stays balanced.

---

## Sales — POS, Registers, Orders & Sync

### POS sale (cart) — `/admin/pos`

![POS terminal](docs/screenshots/pos/terminal.png)
- **Purpose:** Build a retail cart (scan/tap products, per-line and order discounts, attach customer, hold/resume) and start checkout.
- **Access:** SELLING_ROLES = owner, admin, manager, cashier (enforced by `posCheckout`).
- **Backend:** `posCheckout` (`src/lib/pos.functions.ts`) → `recordPosSale` (`src/lib/pos.server.ts`) → `record_pos_sale` RPC → `orders`, `order_items`, `payments`, `receipts`, `inventory_movements`/`inventory_balances`, `journal_entries`, `cash_movements`.

**Order-level fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Register (header select) | `register_id` | uuid | No | first open/first register | must be a shop register | `null` allowed |
| (scan box) | scan/SKU | text | — | — | matches barcode or SKU | Enter adds item; not persisted |
| Search (F2) | search | text | No | — | client filter, ≤60 shown | not persisted |
| Category chips | category | text | No | `all` | derived from catalog | client filter |
| Guest / Customer (F4) | `customer_id` | uuid | No | null (Guest) | select from `customers` | attach/detach |
| Order discount (F6) | `cartDiscount` | number | No | 0 | capped at `gross − lineDiscounts` | spread evenly across lines on submit |
| — | `local_uuid` | text(≤80) | auto | `crypto.randomUUID()` | idempotency key | dedup on `(shop_id, local_uuid)` |
| — | `notes` | text(≤500) | No | null | — | in schema; not sent by POS online path |
| — | `branch_id` | uuid | No | null | — | not in POS UI |
| Subtotal | `gross` | number | auto | — | Σ price×qty | `auto` |
| Discounts | line+order disc | number | auto | — | Σ line.discount + orderDiscount | `auto` |
| Tax | `taxTotal` | number | auto | — | per line: the product's tax rate, else the shop default; inclusive or exclusive | `auto`; shown when > 0, labelled **Incl. tax** for tax-inclusive rates |
| Total | `total` | number | auto | — | net subtotal + tax | `auto`; equals the authoritative server `priceCart` total — the client prices each line with the **same** shared helpers (`resolveLineTax` / `priceLine` / `sumPricedLines`), so the cashier collects exactly what the server posts |

**Cart lines** (UI `Line`; server `posLine`)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (product) | `variant_id` | uuid | Yes | — | valid variant | one row per variant |
| Qty (−/input/+) | `quantity` | number | Yes | 1 | 0.001–9999; ≤0 removes line | numeric(14,3) |
| Disc. | `discount` | number | No | 0 | ≥0, max 1,000,000 | submitted as `line.discount + perLineOrderDiscount` |
| line amount | line total | number | auto | — | `price×qty − discount` | `auto` |

**Offline snapshot** (sent only on offline queue / sync retry — `client_snapshot`): `device_id` (text ≤80, marks `orders.is_offline`), `local_seq` (int), `offline_created_at` (ISO), and immutable `client_snapshot.{lines, subtotal, discount_total, tax_total, total}` — the "as-charged" record the server reconciles against; variance → `pos_sync_conflicts`.

**Keyboard shortcuts:** F2 Search · F4 Customer · F6 Order discount · F8 Hold · F9 Pay · Esc Cancel/close (↑/↓/←/→ move the grid highlight; Enter in search adds the highlighted item). Hold/Resume held sales are client-only state (no backend).

**Connections**
- **Selects / references:** Variant/Product (catalog + `inventory_balances`), Customer (`customer_id`), Register (`register_id`); open `cash_sessions` resolved server-side.
- **Feeds / used by:** completed sale → `order_items`, `payments`, `inventory_movements`, `receipts`, `journal_entries` (Dr Cash 1000/Bank 1100; Cr Revenue 4000 + Tax 2100; Dr COGS 5000 / Cr Inventory 1200), `cash_movements`; low-stock → `ai_alerts`.
- **Cross-form effects:** server is the pricing authority (recomputes via `priceCart`); payment total must equal order total within currency tolerance; duplicate `local_uuid` returns the existing order; offline/failed sales queue to `/admin/sync`.

---

### Payment screen — `/admin/pos` (Pay modal, F9)
- **Purpose:** Allocate split payments across methods and compute change, then complete the sale.
- **Access:** SELLING_ROLES. **Backend:** same as POS Sale → `payments` (+ `cash_movements` for cash).

**Split-payment rows** (UI `PaymentRow`; server `payments[]`)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Method tiles / Pay N select | `method_code` | text(≤40) | Yes | `cash` | cash, card, bank, credit, other | cash→acct 1000, else→bank 1100 |
| Pay N amount | `amount` | number | Yes | subtotal (first row) | ≥0; only rows >0 submitted | numeric(14,2) |
| — | `reference` | text(≤120) | No | — | — | in schema; not in POS UI |
| — | `status` | text | auto | `captured` | — | `auto` |
| Allocated | `allocated` | number | auto | — | Σ amounts | `auto` |
| Remaining | `remaining` | number | auto | — | subtotal − allocated | `auto` |
| Change due | `changeDue` | number | auto | — | max(0, allocated − subtotal) | `auto`; returned as `change` |

≥1 payment required; "Split payment" adds a row prefilled with the remaining. Complete is enabled only when fully allocated (allocated ≥ subtotal, some amount >0, subtotal >0).

**Credit (pay later) — udhaar.** When *Settings → Customers & credit* has credit sales on and the signed-in user's role is permitted (`credit_roles`), the Pay screen shows a **Credit sale (udhaar / pay later)** toggle. With it ticked the button becomes **Complete credit sale** and the sale can post with **partial or zero** tender — the unpaid remainder is booked to the customer's account (**Accounts Receivable**) and the order is marked `partial` or `unpaid`.
- **A customer is required** — a credit sale can't complete as a guest (you're prompted to attach one first).
- The customer's **credit limit is enforced**: their existing outstanding balance + this sale must stay within their `credit_limit` (`0` = unlimited). A **manager** sees an **Override the customer's credit limit** checkbox to push past it.
- Credit sales are **online-only** — they're server-validated and are never queued offline. Settle the balance later via *Sales → Record payment*.

---

### Open cash session — `/admin/pos` header & `/admin/registers`
- **Purpose:** Open a register drawer with an opening float.
- **Access:** any active member; non-managers must be assigned to the register.
- **Backend:** `openCashSession` (`src/lib/cash.functions.ts`) → `cash_sessions` + `cash_movements` (`opening`).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (register select) | `registerId` | uuid | Yes | — | shop register | — |
| Opening float | `openingBalance` | number | Yes | 0 | ≥0 | rejected if register already has an open session |
| — | `status` | text | auto | `open` | — | `auto` |
| — | `opened_by` / `opened_at` | uuid / ts | auto | caller / now | — | `auto` |

---

### Cash movement (in/out/expense) — `/admin/registers` (session panel)
- **Purpose:** Record cash added to / removed from an open drawer, or a cash expense.
- **Access:** any active member; UI enables only for managers or the assigned opener.
- **Backend:** `recordCashMovement` (`cash.functions.ts`) → `cash_movements` (+ `recordExpenseAndPost` → expense + `journal_entries` when type=expense).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (session context) | `sessionId` | uuid | Yes | selected session | must be `open` | — |
| Type | `movementType` | enum | Yes | `cash_in` | cash_in, cash_out, expense | — |
| Amount | `amount` | number | Yes | — | >0 (min 0.01) | numeric |
| Note | `note` | text(≤300) | No | null | — | — |
| — | `categoryId` | uuid | No | null | expense category | only for `expense`; not in registers UI |
| — | `reference_type`/`reference_id` | text/uuid | auto | null | set to `expense`/expenseId | `auto` when expense |

---

### Close cash session — `/admin/pos` header & `/admin/registers`
- **Purpose:** Count the drawer and close the session, computing variance vs expected.
- **Access:** any member; only the opener or a manager may close.
- **Backend:** `closeCashSession` (`cash.functions.ts`) → `cash_sessions` update + `cash_movements` (`closing`) + `ai_alerts` (on discrepancy).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (session context) | `sessionId` | uuid | Yes | selected/open session | must be `open` | — |
| Counted cash | `countedBalance` | number | Yes | 0 | ≥0 | numeric(14,2) |
| Expected | `closing_balance` | number | auto | — | opening + cash_sales − cash_refunds + cash_in − cash_out − expenses | `auto` |
| Difference | discrepancy | number | auto | — | counted − expected | `auto`; alert if abs>0 (critical if >5% of expected) |
| — | `closed_at` / `status` | ts / text | auto | now / `closed` | — | `auto` |

Read-only reconciliation stats: Opening, Cash sales, Refunds, Cash in, Cash out, Expenses, Expected, Counted, Difference.

---

### Register create / edit / assign — `/admin/registers`
- **Access:** MANAGERS = owner, admin, manager. **Backend:** `createRegister` / `updateRegister` / `assignRegisterUser` / `unassignRegisterUser` (`cash.functions.ts`) → `registers`, `register_users`.

**Create/edit**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Name | `name` | text | Yes | — | 1–120 chars | — |
| Code | `code` | text | No | null | ≤40 chars | — |
| — | `branch_id` | uuid | No | null | shop branch | not in UI |
| Active/Inactive (toggle) | `is_active` | boolean | No | true | — | the only field edit UI toggles |

**Assign users** — per-member checkbox writes `register_users` (`registerId` + `userId`, idempotent, unique `(register_id,user_id)`).

**Connections (registers)** — references Register (`register_id`), Branch (`branch_id`), staff from `shop_users`, session (`cash_session_id`). An open register session gates POS selling; movements + sessions feed reconciliation and journal entries; discrepancies raise `ai_alerts`. One open session per register; refunds land back in the originating open session's drawer.

---

### Return / refund — `/admin/orders` (order detail)
- **Purpose:** Return order lines, optionally restock, and issue a refund.
- **Access:** MANAGERS (refunds move money out).
- **Backend:** `posRefund` (`pos.functions.ts`) → `recordReturn` → `record_pos_return` RPC → `returns`, `return_items`, `refunds`, `inventory_movements` (restock), `journal_entries` (reversing), `cash_movements`, `orders` status.

**Header fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (order) | `orderId` | uuid | Yes | — | order in shop | — |
| Reason | `reason` | text(≤300) | No | null | dropdown | UI: **Reason** dropdown (e.g. "Customer changed mind") |
| Refund method | `method_code` | text(≤40) | No | `cash` | cash refunds re-enter open drawer | UI: **Refund method** selector (Cash …) |
| Refund total | `total` | number | auto | — | Σ proportional line totals; must be >0 | `auto` |
| Return number | `return_number` | text | auto | — | per-shop unique | `auto` |

**Return lines** (server `items[]`)

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (line) | `order_item_id` | uuid | Yes | — | belongs to order | — |
| Qty | `quantity` | number | Yes | 0 | ≥0; clamped to remaining (sold − already returned) | UI: per-line **Return quantity** input (partial returns supported) |
| Restock | `restock` | boolean | No | true | toggles inventory return | UI: **Restock returned items** toggle (default on); restocked units reverse COGS |

The `admin.orders.tsx` **Return items** modal surfaces per-line **return quantity** (partial returns), a **Reason** dropdown, a **Refund method** selector, and a **Restock returned items** toggle; the live **Refund ≈** estimate updates as you set quantities, and **Confirm return** posts the reversing entry. Verified example: a partial return of one line on a ₨16,399 sale created **RET-000001** (₨3,499 refund), set the order payment status to **partial**, restocked the unit, and reduced Revenue −₨3,499 + COGS −₨1,800 in the ledger (still balanced; A = L + E holds).

---

### Sales — list & detail — `/admin/sales`
- **Purpose:** Browse **every** counter / online / AI sale in one place, open a sale for full detail, and **record a payment** to settle a credit/partial balance.
- **Access:** view a sale's detail = `owner`, `admin`, `manager`, `cashier`, `accountant` (`getSaleDetail`); **Record payment** = `owner`, `admin`, `manager`, `accountant` (`recordSalePayment`).
- **Backend:** list via a direct `orders` read; detail via `getSaleDetail`; payment via `recordSalePayment` (`src/lib/orders.functions.ts`) → `record_order_payment` RPC → `payments` (+ updates `orders.paid_total`/`payment_status`).

**List filters**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Search | `search` | text | No | "" | matches `order_number` / `contact_email` / `contact_phone` | — |
| Channel | `channel` | select | No | `all` | all / pos / online / ai | — |
| Payment | `payment` | select | No | `all` | all / paid / partial / unpaid (credit) / refunded | — |
| From / To | `dateFrom` / `dateTo` | date | No | — | filter on `created_at` | — |

Each row shows **Sale # · Date · Channel · Customer · Payment · Total · Outstanding** (Total − Paid). **View** opens a read-only detail modal: line items (qty / unit / discount / tax / total), the subtotal → discount → tax → shipping → **Total** breakdown, **Paid** vs **Outstanding**, the **tenders** (payments), plus customer, register, cashier and any returns.

**Record payment** — shown in the detail modal only while a sale still has an outstanding balance (a credit / partial sale):

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Amount | `amount` | number | Yes | — | > 0 | **Full balance** button fills the outstanding amount |
| Method | `method_code` | select | Yes | `cash` | cash / card / bank / other | — |

Recording a payment increments `paid_total`, moves the sale to `partial` / `paid`, and adds a tender row — the same settlement path used to clear udhaar balances taken at the till.

---

### Offline sync — queue & conflict resolution — `/admin/sync`
- **Purpose:** Retry queued offline sales and resolve server-side price/tax/rejected conflicts (accept / reject / retry).
- **Access:** viewing = any member; **resolve/discard/retry = owner, admin, manager** (DESTRUCTIVE_ROLES).
- **Backend:** `syncOfflineSales` (`pos.functions.ts`, batch ≤50); `listSyncConflicts` / `resolveSyncConflict` (`sync.functions.ts`) → `pos_sync_conflicts` (+ `recordPosSale` on retry of `rejected`).

**Conflict resolution fields**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (row) | `conflictId` | uuid | Yes | — | conflict in shop | — |
| Retry / Accept / Reject | `action` | enum | Yes | — | accept \| reject \| retry | accept→resolved; reject→rejected; retry→re-post (rejected) or bump `attempts` |
| Note | `note` | text(≤500) | No | null | disabled unless status pending/retrying | stored in `resolution` |
| Kind | `kind` | text | auto | — | price_variance / tax_variance / rejected | `auto` |
| Status | `status` | text | auto | `pending` | pending/retrying/resolved/rejected | `auto` |
| Offline vs Server total | snapshot.total / server_state.total | number | auto | — | display only | `auto` |
| Attempts | `attempts` | int | auto | 1 | +1 per retry | `auto` |

**Local device queue** ("On this device" / "Rejected on this device", IndexedDB via `src/lib/offline`, not a DB table): Retry resends the `PendingSale` payload (incl. `device_id`, `local_seq`, `offline_created_at`, `client_snapshot`) via `syncOfflineSales`; Discard (manager-only) drops the local pending sale/conflict.

**Connections**
- **Selects / references:** conflicts reference `register_id`, `branch_id`, `device_id`, `local_uuid`, `cashier_id`; payload holds the original sale + snapshot, `server_state` holds the server re-price.
- **Feeds / used by:** retrying a `rejected` conflict/queued sale calls `record_pos_sale` (idempotent on `local_uuid`); variance conflicts already have a posted order and only log the variance.
- **Cross-form effects:** every offline sale failing validation is recorded as a `rejected` conflict rather than lost; the server never overwrites a posted order — it honours the as-charged snapshot and records the difference here.

**Tables owned:** `orders` (FKs shop_id→shops, branch_id→branches, register_id→registers, cash_session_id→cash_sessions, customer_id→customers; unique `(shop_id, order_number)`, partial-unique `(shop_id, local_uuid)`), `order_items` (order_id→orders, variant_id→product_variants), `payments` (order_id→orders), `receipts` (order_id→orders; unique `(shop_id, receipt_number)`), `returns` (order_id→orders; unique `(shop_id, return_number)`), `return_items` (return_id→returns, order_item_id→order_items, variant_id→product_variants), `refunds` (order_id→orders, return_id→returns), `registers` (branch_id→branches), `register_users` (register_id→registers; unique `(register_id, user_id)`), `cash_sessions` (register_id→registers), `cash_movements` (cash_session_id→cash_sessions, register_id→registers; type sale/refund/expense/cash_in/cash_out/opening/closing), `cash_accounts` (account_id→accounts), `pos_sync_conflicts` (branch_id→branches, register_id→registers; unique `(shop_id, local_uuid)`).

---

## Storefront Editor & AI (Admin)

All Storefront Editor forms load via `getAdminStorefrontFn` → `getAdminStorefront` (`src/lib/storefront-admin.server.ts`), returning `{ shop, settings, banners, sections, navigation, policies, pages, themes }` for the active shop. All writes require **owner / admin / manager** (`assertManager`). The editor is a single route with tabs (`theme`, `banners`, `sections`, `navigation`, `pages`, `policies`, `contact`, `domains`) plus an opt-in live-preview iframe of `/store/<slug>?preview=1`.

**Draft vs Publish model:**
- **Theme tab** writes go into `storefront_settings.draft` (a JSON blob) via `saveSettingsDraftFn` and never touch live columns until publish.
- **Banners / Sections / Navigation** rows are written with `is_draft = true` on every save.
- **Publish** (`publishStorefrontFn`): copies each key in `SETTINGS_DRAFT_KEYS` from `draft` into live columns, sets `is_published = true`, clears `draft = {}`, and flips `is_draft = false` on all draft banner/section/nav rows.
- **Discard draft** (`discardStorefrontDraftFn`): clears `draft = {}` and **deletes** banner/section/nav rows still `is_draft = true`.
- **Pages, Policies, Contact/Branding** write straight to live (no draft staging); Pages/Policies have their own `is_published` flag.

---

### Theme — `/admin/storefront` (tab: Theme)
- **Purpose:** Colours, layout variant, font, hero, announcement bar, social links, SEO meta.
- **Backend:** `saveSettingsDraftFn` (`src/lib/storefront-admin.functions.ts`) → `storefront_settings.draft` (JSON merge). "Reset to theme default" calls the same fn with only colour/font keys.

Saved as one draft patch. Colour fields initialise from the theme's own palette when the shop hasn't genuinely customised them (`customColor` / `THEME_DEFAULT_SETTINGS`, `src/lib/storefront-theme.ts`).

| UI label | Field key (patch) | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Homepage variant | `homepage_variant` | select | — | `"index"` | from `themes[0].variants[].code`, else index/index-2…index-5 | — |
| Primary | `primary_color` | color/hex | — | `#0A69D8` | free hex + picker | legacy default `#ff6a00` treated as "not customised" |
| Secondary | `secondary_color` | color/hex | — | `#333333` | — | legacy default `#111111` |
| Accent | `accent_color` | color/hex | — | `#ffa500` | — | legacy default `#f5a623` |
| Text | `text_color` | color/hex | — | `#333333` | — | — |
| Background | `background_color` | color/hex | — | `#ffffff` | — | — |
| Font family (optional) | `font_family` | text | — | `""` | empty → saved as `null` (keeps theme typography) | — |
| Hero title | `hero.title` | text | — | `""` | packed into `hero` JSON | — |
| Hero subtitle | `hero.subtitle` | text | — | `""` | — | — |
| Hero image | `hero.image_url` | image URL | — | `""` | `AdminImageUpload` folder `cms` | — |
| Hero CTA label | `hero.cta_label` | text | — | `""` | — | — |
| Show announcement bar | `announcement_bar.enabled` | checkbox | — | false | — | — |
| Announcement text | `announcement_bar.text` | text | — | `""` | — | — |
| Facebook / Instagram / TikTok / WhatsApp | `social_links.*` | text | — | `""` | packed into `social_links` JSON | — |
| Meta title | `seo.title` | text | — | `""` | packed into `seo` JSON | feeds storefront `<title>` |
| Meta description | `seo.description` | text | — | `""` | — | feeds meta description |
| Meta keywords | `seo.keywords` | text | — | `""` | — | — |

**Reset to theme default** patches the five colours + `font_family` to `THEME_DEFAULT_SETTINGS`, leaving hero/announcement/social/SEO untouched. Still requires Publish. Persisted draft keys (`SETTINGS_DRAFT_KEYS`): homepage_variant, primary/secondary/accent/text/background_color, font_family, announcement_bar, header, footer, hero, social_links, seo, theme_code (`header`/`footer`/`theme_code` are draftable but not in this UI).

---

### Banners — `/admin/storefront` (tab: Banners)
- **Backend:** `saveBannerFn` / `deleteBannerFn` → `storefront_banners`. Every save sets `is_draft = true`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| — | `id` | uuid | — | null (new) | present → update | — |
| Placement | `placement` | select | Yes | `"hero"` | hero, promo | — |
| Position | `position` | number | Yes | `(count+1)*10` | int ≥ 0 | ↑/↓ buttons |
| Title | `title` | text | — | `""` | nullable | "(untitled)" if blank |
| Subtitle | `subtitle` | text | — | `""` | nullable | — |
| Banner image | `image_url` | image URL | — | `""` | folder `cms` wide | — |
| Link URL | `link_url` | text | — | `""` | nullable | — |
| Visible | `is_visible` | checkbox | Yes | true | — | — |
| — | `is_draft` | boolean | auto | true on save | — | cleared on publish |

---

### Sections — `/admin/storefront` (tab: Sections)
- **Backend:** `saveSectionFn` / `deleteSectionFn` → `storefront_sections`. Save sets `is_draft = true`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| — | `id` | uuid | — | null | — | — |
| Page slug | `page_slug` | text | Yes | `"home"` | which storefront page | — |
| Section type | `section_type` | select | Yes | featured_categories | featured_categories, featured_products, flash_deals, product_section, promotional, footer | — |
| Title | `title` | text | — | `""` | nullable | — |
| Position | `position` | number | Yes | `(count+1)*10` | int ≥ 0 | — |
| Visible | `is_visible` | checkbox | Yes | true | — | — |
| Section image | `config.image_url` | image URL | — | — | stored in `config` JSON | — |
| Config (JSON) | `config` | JSON textarea | Yes | `{}` | must parse ("Config must be valid JSON") | free-form block config |
| — | `is_draft` | boolean | auto | true on save | — | — |

---

### Navigation — `/admin/storefront` (tab: Navigation)
- **Backend:** `saveNavItemFn` / `deleteNavItemFn` → `storefront_navigation`. Save sets `is_draft = true`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| — | `id` | uuid | — | null | — | — |
| Location | `location` | select | Yes | `"header"` | header, footer | — |
| Label | `label` | text | Yes | `""` | min 1 | — |
| URL (relative) | `url` | text | Yes | `""` | empty allowed; col NOT NULL | rendered as `/<url>` |
| Position | `position` | number | Yes | `(count+1)*10` | int ≥ 0 | — |
| Visible | `is_visible` | checkbox | Yes | true | — | — |
| — | `parent_id` | uuid | — | null | self-FK for nesting | not in UI |
| — | `is_draft` | boolean | auto | true on save | — | — |

---

### Pages — `/admin/storefront` (tab: Pages)
- **Backend:** `savePageFn` / `deletePageFn` → `storefront_pages` (live, no draft staging).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| — | `id` | uuid | — | null | — | — |
| Slug | `slug` | text | Yes | `""` | min 1 | unique `(shop_id, slug)` |
| Title | `title` | text | Yes | `""` | min 1 | — |
| Published | `is_published` | checkbox | Yes | false | col default true | "· unpublished" in list |
| (content) | `content` | textarea | Yes* | `""` | empty allowed | body text |

---

### Policies — `/admin/storefront` (tab: Policies)
- **Backend:** `savePolicyFn` → `storefront_policies` (upsert on `shop_id, policy_type`, live).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Page | `policy_type` | select | Yes | `"privacy"` | privacy, terms, return, shipping, faq | selects which row is edited |
| — | `title` | text | auto | from `POLICY_TYPES` map | min 1 | derived, not a visible input |
| (content) | `content` | textarea | Yes* | existing / `""` | — | — |
| Published | `is_published` | checkbox | Yes | true | — | — |

`POLICY_TYPES` titles: Privacy Policy, Terms & Conditions, Return Policy, Shipping Policy, FAQ.

---

### Branding & Contact — `/admin/storefront` (tab: Contact)
- **Backend:** `saveContactInfoFn` → **`shops`** table (not a storefront_* table). Written live.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Store name | `name` | text | — | `shop.name` | only written if non-empty | updates `shops.name` |
| Tagline | `tagline` | text | — | `shop.tagline` | nullable | — |
| Store logo | `logo_url` | image URL | — | `shop.logo_url` | folder `branding` | header + receipts |
| Favicon | `favicon_url` | image URL | — | `shop.favicon_url` | folder `branding` | browser tab icon |
| Phone | `phone` | text | — | `shop.phone` | nullable | — |
| Email | `email` | text | — | `shop.email` | nullable | — |
| WhatsApp | `whatsapp` | text | — | `shop.whatsapp` | nullable | — |
| Address | `address` | text | — | `shop.address` | nullable | — |

**Storefront Connections**
- **Selects / references:** Sections/banners reference categories & products (via `config` JSON / links); Navigation URLs point at storefront customer pages; Pages & Policies render as customer info pages.
- **Feeds / used by:** all published rows drive the public storefront at `/store/<shop.slug>`; `seo` feeds meta tags; theme colours/font inject storefront CSS; branding feeds header + receipts. Public reads are RLS-gated to `is_visible AND NOT is_draft`.
- **Cross-form effects:** Theme + banner/section/nav changes stay hidden until **Publish**; currency/language/timezone are managed in `/admin/settings`, not here.

---

### Domains (connect your own domain) — `/admin/storefront` (tab: Domains)
- **Purpose:** Connect your own domain(s) (e.g. `trendies.pk`) so your storefront is served at **clean root URLs** on your own brand — `https://trendies.pk/`, `/product/<slug>`, `/cart`, `/checkout` — with **no** `/store/<slug>/` prefix. The original `/store/<shopSlug>/*` links keep working.
- **Access:** owner / admin only (`requireRole(..., ADMINS)`).
- **Backend:** `listDomainsFn` / `addDomainFn` / `verifyDomainFn` / `setPrimaryDomainFn` / `removeDomainFn` (`src/lib/domains.functions.ts`) → `domains.server.ts` → **`shop_domains`** table.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Add a domain | `hostname` | text | Yes | — | 3–253 chars; one shop per host (case-insensitive) | e.g. `trendies.pk` or `www.trendies.pk` |
| — | `verification_status` | enum | auto | `pending` | pending / verified / failed | status badge per row |
| — | `is_primary` | boolean | auto | false | one primary per shop | set via **Make primary** (verified only); others redirect to it |

**Merchant flow**
1. **Add the domain** — Storefront → **Domains**, type the host, click **Add domain** (lists as **Pending**).
2. **Create the DNS records** at your registrar: an **A record** (`@` → `185.158.133.1`) plus the **`_lovable` TXT** verification record — use the exact values shown on screen.
3. The platform **verifies ownership and issues the SSL certificate** (the free `https://` padlock).
4. Once the domain resolves to your store, click **Mark verified** — it switches to **Verified** and your storefront goes live on it.
5. *(Optional)* If you added more than one (e.g. apex + `www`), click **Make primary**; secondary domains **redirect** to the primary.
6. *(Optional)* **Serves** — each domain has a **Serves** selector (shown as soon as the domain is added, even before DNS verification) choosing WHAT it serves at its root: **Online store** (default — the storefront, as above) or **Restaurant website** (the `/dine` site from Restaurant → Website). A restaurant-assigned domain opens the restaurant site at `/` and is its own home — it is **not** redirected to the store's primary domain. Changing it saves immediately and is recorded in the audit log (`shop_site.domain_assigned`).

Unknown / unverified hosts never resolve to a shop, and one domain maps to exactly one shop. Full step-by-step for Namecheap is in **[CONNECT_DOMAIN_NAMECHEAP.md](./CONNECT_DOMAIN_NAMECHEAP.md)**, and the same guide is in-app at **`/admin/domain-setup`**.

---

## AI (Admin) — `/admin/ai`

![AI business assistant](docs/screenshots/ai/business-assistant.png)

Three tabs: **Assistant** (chat + right-rail panels: reports, pending actions, action history, permissions, alerts), **Knowledge base**, **Business memory**. Most AI writes require **owner / admin / manager**; chat itself only needs active shop membership.

### Assistant chat — `/admin/ai` (tab: Assistant)
- **Backend:** `askAssistant` (`src/lib/ai.functions.ts`) → `aiChat` (agent `"assistant"`) → `ai_conversations`, `ai_messages`, `ai_tool_calls`, may insert `ai_action_approvals`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| message box | `message` | text | Yes | `""` | 1–2000 chars | Enter to send |
| — | `shopId` | uuid | auto | current shop | — | — |
| — | `conversationId` | uuid | auto | null first turn, then returned id | — | threads the conversation |
| — | `locale` | text | auto | `shop.default_locale` | ≤5 | — |

Reports: **Morning briefing** (`getBriefing`) and **End-of-day summary** (`getEndOfDay`), input `{shopId}` only.

**The assistant can take actions, not just answer.** Ask it to **create a purchase order**, **adjust inventory**, **record an expense or payment**, or **draft a message**; it resolves the records it needs (supplier, product, amounts) and then either runs immediately or queues for approval depending on that capability's **mode** in *AI permissions* (disabled / approval / automatic). Approval-mode actions appear under **Pending AI actions**; on **Approve** the action is actually executed and its result is posted back into the chat (see *Permission matrix* and *Approval queue*). Actions never exceed the acting user's role limits.

---

### Permission matrix — `/admin/ai` ("AI permissions" panel)
- **Purpose:** Per-capability mode governing whether the AI may act.
- **Backend:** direct Supabase update on `ai_permissions.mode`; server-side action execution re-checks `has_shop_role`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| (agent label) | `agent` | enum | auto | seeded | `ai_agent` = salesman \| assistant | display only |
| (capability label) | `capability` | text | auto | seeded | — | display only |
| mode dropdown | `mode` | select | Yes | col default `disabled` | disabled \| approval \| automatic | the only editable field |
| — | `id`, `shop_id` | uuid | auto | — | unique `(shop_id, agent, capability)` | — |

**Seeded capabilities** (one `ai_permissions` row per shop): salesman → view_products / view_inventory / view_prices / recommend_products / add_to_cart / track_order (automatic), create_order (approval). assistant → view_sales / view_inventory / view_purchases / view_expenses / view_accounting / view_customers / suggest_reorders / generate_reports (automatic), create_purchase_order / adjust_inventory / record_expense / issue_refund (approval).

---

### Approval queue — `/admin/ai` (Pending AI actions / Action history)
- **Backend:** read `ai_action_approvals` (limit 30, newest first); decide via `decideAiApproval` → on approve runs `executeAiAction`, writes result back, posts an `ai_messages` summary.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Approve / Reject | `decision` | button | Yes | — | approved \| rejected | — |
| — | `approvalId` | uuid | auto | row id | — | — |
| — | `shopId` | uuid | auto | current shop | — | — |

Row display fields: `action`, `payload` (summarised: supplier, items, total, amount, quantity, variant, reason, note), `status` (`approval_status`: pending/approved/rejected/executed, plus app-written `failed`), `created_at`, `executed_at`, `result`, `error`, `requested_by`, `decided_by`, `decided_at`, `conversation_id`.

---

### Alerts — `/admin/ai` ("Alerts" panel)
- **Backend:** `listAlerts` / `refreshAlerts` (`src/lib/ai-insights.functions.ts`) → `ai_alerts`. Dismiss = direct update `is_read=true`. Columns: `alert_type`, `severity` (info/warning/critical), `title`, `message`, `data` (JSON), `is_read`, `created_at`.

---

### Knowledge base — `/admin/ai` (tab: Knowledge base)
- **Backend:** `addKnowledgeDocumentFn` / `listKnowledgeDocumentsFn` / `deleteKnowledgeDocumentFn` → `ai_knowledge_documents` (+ chunks into `ai_knowledge_chunks`).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Title | `title` | text | Yes | `""` | 1–200; file upload falls back to `file.name` | — |
| (content) | `content` | textarea | Yes | `""` | 1–200000 | markdown/CSV/plain |
| (file picker) | `content` (from file) | file | — | — | `.txt, .md, .csv` read as text | — |
| — | `source` | text | — | `"manual"` | ≤100 | not in UI |
| — | `docType` | text | — | server "text" | col `doc_type` default `policy` | not in UI |
| — | `is_active` | boolean | auto | true | — | — |

`content` is split by `chunkText` (~800 chars, 120 overlap) into `ai_knowledge_chunks` (each with a generated `search` tsvector). Returns `{documentId, chunkCount}`. List row shows title, chunk_count (auto), source, created_at.

---

### Business memory — `/admin/ai` (tab: Business memory)
- **Backend:** `listBusinessMemoryFn` / `createBusinessMemoryFn` / `updateBusinessMemoryFn` / `setBusinessMemoryActiveFn` / `decideBusinessMemoryFn` (`src/lib/ai-memory.functions.ts`) → `ai_business_memory`.

**Add memory**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Scope | `scope` | text | Yes | `""` | 1–50; col default `general` | e.g. policy, supplier, hours |
| (content) | `content` | textarea | Yes | `""` | 1–4000; sanitised (cards/tokens/emails redacted) | — |
| — | `source` | enum | auto | `"user"` | — | user-created |
| — | `status` | enum | auto | `"approved"` (user) / `pending` (AI) | — | — |
| — | `confidence` | numeric | auto | 1 (AI proposals 0.6) | — | — |
| — | `is_active` | boolean | auto | true | — | — |

**Edit** `id` + `scope?`/`content?`. **Toggle active** `id` + `isActive`. **Decide AI-proposed** `id` + decision (approved\|rejected) — only affects `source='ai' AND status='pending'`; approve also sets `is_active=true`. The UI splits into AI-proposed (pending → Approve/Reject) and Approved (Edit / Activate-Deactivate). AI reads (`activeBusinessMemory`) only include `is_active=true AND status='approved'`.

**AI Connections**
- **Permissions gate actions:** a capability's `mode` decides disabled (blocked) / approval (queued into `ai_action_approvals`) / automatic (executed). Approve re-verifies role + mode server-side before `executeAiAction`.
- **Knowledge → chunks:** documents are chunked into `ai_knowledge_chunks`, retrieved by shop-scoped full-text to ground chat answers.
- **Memory needs approval:** AI-proposed memory stays pending/inactive until a manager approves.
- **Conversations/messages/tool-calls** record the chat; approval decisions post follow-up `ai_messages`.

**Tables owned:**
- **Storefront:** `storefront_settings` (unique `shop_id`; live cols + `draft` jsonb; `theme_code` default `zenis`), `storefront_themes` (global, `variants` jsonb, no shop FK), `storefront_pages` (unique `(shop_id, slug)`), `storefront_sections` (`is_draft`), `storefront_navigation` (self-FK `parent_id`; `is_draft`), `storefront_banners` (`is_draft`), `storefront_policies` (unique `(shop_id, policy_type)`). *Branding/Contact writes target `shops`.*
- **AI:** `ai_permissions` (unique `(shop_id, agent, capability)`; `agent`=`ai_agent`), `ai_conversations` (`customer_id`→customers), `ai_messages` (`conversation_id`→ai_conversations), `ai_tool_calls` (`conversation_id`→ai_conversations), `ai_action_approvals` (`conversation_id`→ai_conversations; `status`=`approval_status`), `ai_knowledge_documents`, `ai_knowledge_chunks` (`document_id`→ai_knowledge_documents; generated `search` tsvector), `ai_alerts`, `ai_business_memory` (`scope`/`source`/`status`/`confidence`/`is_active`; RLS owner/admin/manager).

---

## Storefront — Customer Forms

![Storefront home](docs/screenshots/storefront/home.png)

Auth pages (sign in / up / forgot) call Supabase Auth directly from the browser (`supabase.auth.*`), not a server fn. **Cart, wishlist, and compare are client-only**, persisted in `localStorage` (`zenis-store:<shopSlug>`) via `useStorefrontCart` (`src/lib/cart-store.ts`); the DB `carts` / `cart_items` / `wishlists` tables are **not written** by any storefront route. A `customers` row is created lazily on first authenticated account/checkout access (`getOrCreateCustomer`), or by `placeOnlineOrder` for guests.

---

### Customer sign in — `/store/$shopSlug/sign-in`
- **Purpose:** Authenticate a returning shop-scoped customer (password or Google OAuth).
- **Access:** guest (redirects to `account` if already signed in).
- **Backend:** `supabase.auth.signInWithPassword` / OAuth `google` / `supabase.auth.resend` → `auth.users`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Email | `email` | text | Yes | `""` | — | — |
| Password | `password` | password | Yes | `""` | — | — |
| (Continue with Google) | — | button | — | — | OAuth redirect to `sign-in` | — |
| (Resend confirmation email) | — | button | — | — | shown only when error is "not confirmed" | `auth.resend({type:"signup"})` |

**Connections** — on success navigates to `account`; session drives `getCustomerAccount`/checkout prefill; account is scoped per shop (metadata `shop_slug`).

---

### Customer sign up — `/store/$shopSlug/sign-up`

![Storefront shop](docs/screenshots/storefront/shop.png)

- **Access:** guest. **Backend:** `supabase.auth.signUp` / OAuth `google` → `auth.users`; app `customers` row created later, lazily.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Full name | `fullName` | text | No | `""` | → `user_metadata.full_name` | — |
| Email | `email` | text | Yes | `""` | non-empty (trimmed); Supabase email format | — |
| Password | `password` | password | Yes | `""` | min length 6 (client) | — |
| (shop scope) | `shop_slug` | auto | auto | `shop.slug` | — | → `user_metadata.shop_slug` |
| (Continue with Google) | — | button | — | — | OAuth redirect to origin | — |

**Connections** — confirmed session → `account`; later `getOrCreateCustomer` inserts `customers` (shop_id, user_id, email). Duplicate-email surfaced as "already exists".

---

### Forgot password — `/store/$shopSlug/forgot-password`
- **Access:** guest. **Backend:** `supabase.auth.resetPasswordForEmail` → email; redirects to shared `/reset-password?next=<account>`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Email | `email` | text | Yes | `""` | trimmed | neutral "if that email exists" message |

---

### Product page — add to cart — `/store/$shopSlug/product/$slug`

![Product detail](docs/screenshots/storefront/product-detail.png)
- **Purpose:** Choose a variant + quantity and add to cart (or wishlist/compare).
- **Access:** guest or signed-in. **Backend:** none on add (localStorage `addToCart`/`toggleList`); page data via `getStorefrontProduct`. **No review form exists.**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Options | `variantId` | select (buttons) | Yes if >1 variant | default variant (`is_default`) else first active | active variants only | hidden when single variant |
| Quantity | `quantity` | number | Yes | 1 | min 1; coerced `max(1, n)` | — |
| Add to cart | — | button | — | — | disabled if no variant or out of stock | writes `CartLine` to localStorage |
| Wishlist (heart) | — | button | — | — | toggles wishlist list | — |
| Compare | — | button | — | — | toggles compare list | — |

**CartLine written per add:** `variant_id`, `quantity`, `name` (product + variant), `sku`, `price`, `image` (first product image), `product_slug`.

**Connections** — reads Product / active Variant (price, compare_at_price, track_inventory, stock); cart/checkout/wishlist/compare read the same localStorage store; stock drives disabled state.

---

### Cart — `/store/$shopSlug/cart`

![Cart](docs/screenshots/storefront/cart.png)
- **Access:** guest or signed-in (client-side). **Backend:** none — localStorage. **No coupon field.**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Item | `name`,`sku`,`price` | display | — | — | — | read-only |
| Quantity | `quantity` | number | Yes | current | min 0; `<=0` removes line | — |
| Total | — | auto | — | `price × quantity` | — | computed |
| (trash) | — | button | — | — | removes line | — |

Totals: Subtotal (auto, `Σ price×qty` client-side); note "Taxes and totals calculated securely at checkout". "Checkout" → checkout, which re-prices server-side.

---

### Checkout — `/store/$shopSlug/checkout`

![Checkout](docs/screenshots/storefront/checkout.png)
- **Purpose:** Capture contact + address + payment method and place an online order.
- **Access:** guest or signed-in (prefills name/email from session).
- **Backend:** `quoteCart` (prices, strips cost) and `placeOnlineOrder` (`src/lib/checkout.functions.ts`) → RPC `record_pos_sale` → `orders`, `order_items`, `payments`, `receipts`, inventory + journal; also `customers`, `ai_alerts`.

**Contact / Billing details**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Full name | `customer.full_name` | text | Yes | prefilled from session | 1–120 chars | UI blocks empty name/email |
| Email | `customer.email` | email | Yes | prefilled from session | valid email | — |
| Phone | `customer.phone` | text | No | `""` | max 40 | — |
| — | `customer.user_id` | uuid | auto | session user id | optional | attaches order to logged-in customer |

**Shipping address sub-form** (`shipping_address`, all optional, stored as JSON on order): `line1` (max 200), `city` (max 80), `postal_code` (max 30), `country` (max 80).

**Payment / meta**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Payment (radio) | `payment_method_code` | radio | Yes | `cod` | cod \| bank \| card | `cod` = unpaid/AR (1150); bank/card = prepaid → paid, Bank (1100) |
| — (notes) | `notes` | text | No | — | max 500 | not in UI (schema-only) |
| — (idempotency) | `client_uuid` | uuid | auto | `crypto.randomUUID()` | max 80 | dedupes retries via `orders.local_uuid` |
| Items | `items[]` | auto | Yes | from cart | `variant_id` uuid, `quantity` 0.001–9999, min 1 line | server re-prices |

**Order summary (auto, from `quoteCart`):** `subtotal`, `tax_total`, `total`, `discount_total`, and generated `order_number` / `receipt_number`.

**Connections**
- **Selects / references:** cart line items → Variants; payment method (hardcoded list, **not** from `payment_methods` table).
- **Feeds / used by:** creates `orders` (channel `online`, status `confirmed`), `order_items`, `payments` (non-COD), `receipts`, decrements inventory, posts journal, `ai_alerts` (new_order) — the same `orders`/`order_items` the admin Orders form reads; `customers` reused or inserted.
- **Cross-form effects:** on success clears cart, navigates to `payment/success?order=<number>`; per-IP rate limit (10/min); idempotent on `client_uuid`.

---

### Track order — `/store/$shopSlug/track-order`

![Track order](docs/screenshots/storefront/track-order.png)
- **Access:** guest (public). **Backend:** `trackOrder` (`src/lib/storefront.functions.ts`) → reads `orders` + `order_items` (contact fields stripped from response).

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Order number | `orderNumber` | text | Yes | `""` | min 1; trimmed | — |
| Email / phone | `contact` | text | Yes | `""` | min 3; must match `contact_email` or `contact_phone` | button disabled until both filled |

Returns status/payment/fulfillment/tracking + items + total (read-only).

---

### Account — Profile — `/store/$shopSlug/account` (tab: Profile)
- **Access:** signed-in customer. **Backend:** `updateCustomerProfile` (`src/lib/customer.functions.ts`) → `customers`.

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| Full name | `full_name` | text | No | customer value | max 120; `""`→null | — |
| Email | `email` | email | No | customer value | max 160; `""`→null | — |
| Phone | `phone` | text | No | customer value | max 40; `""`→null | — |

---

### Account — Addresses — `/store/$shopSlug/account` (tab: Addresses)
- **Access:** signed-in customer. **Backend:** `upsertCustomerAddress` / `deleteCustomerAddress` / `setDefaultCustomerAddress` → `customer_addresses`.

**Address sub-form**

| UI label | Field key | Type | Req | Default | Validation / options | Notes |
|---|---|---|---|---|---|---|
| — | `id` | uuid | auto | — | present = update, absent = insert | — |
| Label | `label` | text | No | `""` | max 60 | — |
| Address line 1 | `line1` | text | No | `""` | max 200 | — |
| Address line 2 | `line2` | text | No | `""` | max 200 | — |
| City | `city` | text | No | `""` | max 100 | — |
| State | `state` | text | No | `""` | max 100 | — |
| Postal code | `postal_code` | text | No | `""` | max 30 | — |
| Country | `country` | text | No | `""` | max 100 | — |
| Phone | `phone` | text | No | `""` | max 40 | — |
| Set as default | `is_default` | checkbox | No | false | — | setting true unsets other defaults first |

**Connections** — saved addresses are the customer's own list (not auto-injected into the checkout form, which has its own inline fields).

---

### Account — Order history / reorder — `/store/$shopSlug/account` (tab: Orders)
- **Access:** signed-in customer. **Backend:** `getCustomerAccount` + `getOrderDetail` → `orders`, `order_items` (scoped to own `customer_id`). Reorder is client-side (`addToCart`).

Read-only list: Order # / Date / Status / Payment / Total (from `orders`); **View** opens the detail modal; **Reorder these items** re-adds each `order_item` (with `variant_id`) to the cart.

---

### Wishlist / Compare — `/store/$shopSlug/wishlist` · `/compare`
- **Access:** guest or signed-in (client-side). **Backend:** none — localStorage `wishlist`/`compare` lists (`toggleList`, `addToCart`). **DB `wishlists` table is not used by the storefront.** Each item shows stored `name`/`sku`/`price`/`image` with Add-to-cart and Remove buttons; Compare shows a Product / SKU / Price table.

---

### Contact — `/store/$shopSlug/contact`
- **Access:** guest. **No submit form** — display-only contact links (address/phone/email/WhatsApp), directing users to the AI chat button. (`vendor.tsx` is likewise a display-only store profile.)

---

### Storefront AI Salesman — chat widget (`/store/$shopSlug`)
- **Purpose:** Help shoppers find products and answer store questions from the storefront chat. **Read-only for the catalog** (no admin data); what it may do is governed by the **salesman** rows in *AI permissions*.
- **Backend:** `askStorefrontAssistant` → `aiChat` (agent `"salesman"`, tools in `src/lib/ai.server.ts`).

It can now:
- **Search / recommend by price, category or what's on sale** — e.g. "bags under 1600", "show me jackets", "what's on sale" (`max_price`, category and `on_sale` filters), returning live **stock** and **discount** info.
- **Surface product reviews** — published customer reviews and the average rating for a product (via `product_reviews`), e.g. when a shopper asks whether an item is any good.
- **Answer store questions** — **accepted payment methods**, the list of **available categories**, contact details, business hours and policies (returns / shipping / privacy / terms / FAQ) via `store_info`.

**Tables owned:** `customers` (`shop_id`→shops, `user_id`→auth.users, unique `(shop_id,user_id)`), `customer_addresses` (`customer_id`→customers), `orders` (`customer_id`→customers, unique `(shop_id,order_number)`, partial-unique `(shop_id,local_uuid)`), `order_items` (`order_id`→orders, `variant_id`→product_variants), `payments` (`order_id`→orders), `receipts` (`order_id`→orders, unique `(shop_id,receipt_number)`). Defined but **unused by these storefront routes** (client-side localStorage instead): `carts`, `cart_items`, `wishlists`. Checkout payment methods are hardcoded (cod/bank/card), not from `payment_methods`. The storefront product page has a **verified-buyer review submission** form (the shopper supplies their order number, which must contain the product); submissions land as `pending` and appear on the storefront only after a manager publishes them from `/admin/reviews`. The AI Salesman surfaces the published rows from the `reviews` table.

---

## Reports

Three read-only report suites, each built the same way: a **report picker**, a **From / To** date range, and **Print** + **Download CSV** actions (`ReportShell`; default range = month-to-date). Because they expose **cost, margin and outstanding balances**, each suite is role-gated. All three read from existing sales / purchasing / inventory data (orders, POs, receipts, movements, balances, ledger) — **no new tables**; each report exports CSV with the columns shown on screen.

### Sales Reports — `/admin/sales-reports`
- **Access:** FINANCE_ROLES — `owner`, `admin`, `manager`, `accountant`.
- **Reports:** Summary · By day · By product · By category · By brand · By payment · By channel · By cashier · By register · By branch · Tax · Returns. The product/category/brand reports show units, revenue, cost, **margin** and margin %.

### Purchase Reports — `/admin/purchasing-reports`
- **Access:** `owner`, `admin`, `manager`, `accountant`, `inventory`.
- **Reports:** PO status · Supplier spend · By product · Goods receipts · Outstanding POs · AP ageing · Cost history.

### Inventory Reports — `/admin/inventory-reports`
- **Access:** `owner`, `admin`, `manager`, `inventory`, `accountant`.
- **Reports:** Valuation · By location · Low stock · Dead stock · Movements · Adjustments · Turnover.

---

## Promotions, Loyalty & Marketing

### Promotion / coupon — `/admin/promotions`
- **Purpose:** A unified, **server-authoritative** discount model. **Promotions** define the discount; **coupons** wrap a promotion with a code and usage limits. Validation and discount computation always run on the server, so the same rule applies identically across POS, online checkout and Restaurant.
- **Access:** MANAGERS (`owner` / `admin` / `manager`); module `marketing`. **Backend:** `marketing.functions.ts` → `marketing.server.ts`. **Tabs:** *Promotions & coupons* (sub-tabs *promotions* / *coupons*) and *Free gifts*.

**New promotion**

| UI label | Type | Req | Default | Notes |
|---|---|---|---|---|
| Name | text | Yes | "" | shown on the discount line |
| Type | select | Yes | Percentage | percentage or fixed-amount |
| Percent (0–100) | number | Yes (percentage) | 0 | discount rate |
| Minimum order | number | No | 0 | only applies at/above this subtotal |
| Max discount (cap, optional) | number | No | — | caps the computed discount |
| Channels | multi-toggle | No | empty = all | `pos` / `online` / `restaurant` |
| Active | toggle | Yes | on | inactive promotions never apply |

**Coupon** — wraps a promotion with a **code** customers enter plus **usage limits**; remaining-use and eligibility checks are enforced server-side at checkout.

### Free-gift campaign — `/admin/promotions` (tab: Free gifts)
- **Purpose:** "Buy X, get Y free" rules. When the cart qualifies (spend threshold or trigger products), eligible gift products are offered; the chosen gift is added at **unit price 0 while its real unit cost still flows to COGS**. Entitlement and gift validation run on the server.
- **Access:** MANAGERS; module `ecommerce`. **Backend:** `free-gift.functions.ts` / `free-gift-admin.server.ts`. Storefront shows a gift picker at checkout; POS applies eligible gifts on the ticket.

### Product reviews & moderation — `/admin/reviews`
- **Purpose:** Verified-purchase product reviews. A shopper submits one from the **storefront product page** ("Write a review") — the form asks for a **rating, order number, name, title and review** and is only accepted if that **order actually contains the product** (verified buyer), so reviews can't be faked. Submissions land as **`pending`** ("your review… will appear once the shop approves it").
- **Access:** MANAGERS (`owner` / `admin` / `manager`) to moderate; module `ecommerce`. **Backend:** `reviews.functions.ts` (`submitReview`, `getProductReviews`, `listReviewsForModeration`, `moderateReview`).
- **Moderate:** the page has **Pending / Published / Rejected** tabs and an **Export CSV**. Each row shows the product, rating, title/body and author with **Publish** / **Reject** actions. **Publishing** makes the review and its star rating appear on the storefront product page and feed the AI Salesman's "is this any good?" answers; **only published reviews are ever public** (anyone may read `status = 'published'`; members read all for moderation). Rejecting keeps it hidden.

### Loyalty program — `/admin/loyalty`
- **Purpose:** One loyalty engine shared by Retail POS, Restaurant and Ecommerce. **Points derive from real sales through an idempotent ledger; balances are never a mutable number.**
- **Access:** MANAGERS to manage the program (some lifecycle actions are `owner`/`admin` only); module `loyalty`.

**Program**

| UI label | Type | Default | Notes |
|---|---|---|---|
| Program enabled | toggle | off | master switch |
| Program name | text | "Loyalty" | shown to customers |
| Tier basis | select | Lifetime spend | how tiers are earned |
| Points earned per 1 spent | number | 1 | earn rate |
| Points to redeem 1 value | number | 100 | redemption rate |
| Minimum redemption (points) | number | 100 | floor to redeem |
| Max redemption (% of order) | number | 50 | caps redemption per order |
| Earn points on tax | checkbox | off | include tax in the earning base |
| Rounding | select | Floor | how fractional points round |

**Tiers** (optional, each with an earning multiplier), **Rewards** (redeemable; discount rewards reuse the checkout discount mechanism), **Point expiration policy** (FIFO, disabled by default; redeemed points are never removed; points before the effective date are grandfathered), **Tier lifecycle** (*Process tier transitions* — idempotent; unchanged tiers create no history), and a **Reward claims** list.

**Earning & redeeming (in practice):** points are **earned automatically** when a sale completes with a **customer attached** — at the POS terminal (tap **Customer** and pick the shopper before taking payment), in Restaurant, or at online checkout — on the earning base (subtotal, or including tax if *Earn points on tax* is on); the running balance appears on the customer's **360 profile**. **Redemption** is done at the **storefront (online) checkout**, where a signed-in customer applies a configured **Reward** (a points-cost discount that reuses the checkout discount mechanism, within *Minimum redemption* and *Max redemption % of order*). The Retail POS terminal **earns** points but does not itself expose reward redemption.

### Marketing campaign — `/admin/campaigns`
- **Purpose:** Target a dynamic segment or static audience over a channel adapter. **Delivery is DEMO until an external provider is configured;** recipient audiences are snapshotted at send time and **consent is enforced**.
- **Access:** MANAGERS; module `marketing`. Related: the **social calendar** below.

### Social calendar — `/admin/social`
- **Purpose:** Plan and schedule social posts on a content calendar. **New post** sets a **Channel** (e.g. Facebook), a **Scheduled time** and a **Caption**, then **Save** adds it to the calendar as a **draft**. **Demo publish** marks the post **"demo published"** — **real publishing is DEMO-only until a social provider is connected** (the calendar clearly distinguishes demo/preview from real publishing; provider credentials are required to post for real).
- **Access:** MANAGERS; module `marketing`.

---

## Manufacturing (BOM & costing)

### Bill of materials — `/admin/manufacturing` (tab: BOMs & costing)
- **Purpose:** Define what it takes to make each manufactured product, then see its **theoretical cost & margin**. One **active BOM per finished variant**; superseding a BOM keeps the older version as history.
- **Access:** INVENTORY_ROLES (`owner` / `admin` / `manager` / `inventory`); module `inventory`. **Backend:** `manufacturing.functions.ts` → `manufacturing.server.ts`; reuses `variant_costs.avg_cost` and the shared unit-conversion helper.

**New BOM** — set **BOM name**, choose the **finished product/variant**, **Output quantity** (units yielded per batch), **Version**, optional **Notes** and an **Active** toggle (the current recipe). *Save the BOM first, then add components.* Add **component** lines:

| UI label | Type | Req | Notes |
|---|---|---|---|
| Component | variant select | Yes | the material/ingredient consumed (multi-select picker with search / category / brand / in-stock filters) |
| Quantity | number | Yes | per one finished unit |
| Unit (UoM) | text | No | defaults to the component's stock unit; converted via unit-conversion |
| Type | select | No | Material / Packaging (classifies the cost bucket) |
| Wastage % | number | No | scrap allowance folded into the rolled-up cost |

An **Additional costs** row adds Labor / Overhead (label + amount). The **Theoretical cost** panel then shows, per batch: **Materials**, **Packaging**, **Additional**, **Total (batch)**, **Unit cost** (rolled up from each component's `avg_cost` × qty, unit-converted, + wastage), the finished variant's **Selling price**, **Gross margin** and **Margin %**. A note flags when any component has no cost / an unsupported unit (the total is then a floor). *A newly created BOM persists as inactive until you toggle Active and save.*

### Production planning — `/admin/manufacturing` (tab: Production planning)
- **Purpose:** Plan production runs against BOMs and preview requirements & cost. Pick a **BOM** + **Produce (units)** → **Calculate** shows the **Unit cost**, **Estimated batch cost** (unit cost × units) and a per-component table of **Required / Available / Shortage / Status** (OK or short). **Production inventory posting is deferred** — planning does **not** consume components or add finished goods to stock ("No inventory is changed" — no stock or GL writes).

---

## Restaurant

Enabled by the three Restaurant modules (**Restaurant POS**, **Restaurant Tables**, **Restaurant Kitchen**). Management screens are gated to MANAGERS (`owner` / `admin` / `manager`); customer-facing ordering surfaces are separate. **Backend:** `restaurant*.functions.ts` (setup rendered by `RestaurantPanel.tsx`).

### Overview — `/admin/restaurant`
- **Purpose:** Operational pulse — cards for **Reservations today**, **Waiting now**, **Kitchen tickets pending** and **Food cost**, plus quick links to Restaurant POS, Delivery, Refunds and Floor & Tables.

### Setup — `/admin/restaurant/<section>` (tabbed)
One page with tabs **Floor & Tables**, **Menu & Modifiers**, **Kitchen Setup**, **Recipes**, **Settings** (URL `section` = `tables` / `menu` / `kitchen` / `recipes` / `settings`). A branch filter scopes everything.

### Restaurant Website — `/admin/restaurant-site`
- **Purpose:** A public restaurant site for your customers — menu showcase, opening info, promotions — **separate from your online store** and from table-QR ordering. It lives at `/store/<your-store>/dine`, and at the **root** of any custom domain you point at it (below). Nothing is public until you flip **Publish**.
- **Backend:** `shop-site.functions.ts` → `shop_site_settings` / `shop_site_banners` / `shop_site_navigation` / `shop_site_pages` (per `site_key`, here `restaurant`). Writes require owner/admin/manager via the Restaurant module's guard; every change is audited (`shop_site.*`).
- **Publish** — on/off switch. Off = a plain "not available" page (hidden from search engines); your store is never affected either way.
- **Branding & hero** — primary/accent colours (`#rrggbb`; leave empty to inherit your storefront theme), announcement bar text, hero headline/subheadline, hero image (**upload**, or paste an https URL), hero button label + link, SEO title/description, social links.
- **Banners** — promo cards under the hero: title, subtitle, image (upload/URL), optional link, position, visible toggle.
- **Navigation links** — header links replace the automatic ones (Menu + your pages); footer links list under *Links*. Label + URL + position + visible.
- **Pages** — simple text pages at `/dine/p/<slug>` (e.g. `opening-hours`): slug, title, content (plain text; a blank line starts a new paragraph), published toggle.
- **The menu is live** — it shows exactly what you curate in Restaurant → **Kitchen Setup** (`restaurant_menu_items`): names, photos, prices and today's availability. There is nothing to duplicate here.
- **Own domain:** Storefront → **Domains** → add/verify the hostname as usual, then set its **Serves** selector to *Restaurant website*. That domain then opens the restaurant site at `/` (your store keeps its own domain). See the Domains section below.

**Floor & Tables**

| Form | Fields |
|---|---|
| Floor | **Name** (e.g. "Ground Floor"), active, branch |
| Table | **Code** (e.g. "T01"), Name (optional), **Capacity** (default 4), Floor, branch, active |

**Menu & Modifiers**

| Form | Fields |
|---|---|
| Modifier group | **Name** (e.g. "Add-ons"), selection type (single / multiple), required, **min / max selections** |
| Modifier option | **Name** (e.g. "Extra Cheese"), **price adjustment** |
| Assign to product | pick a **Product**, then attach modifier **group(s)** |

**Kitchen Setup** — define kitchen **stations** (Code, e.g. "KITCHEN") and route each **product → station** with an optional **prep time** (drives KOT routing on the KDS). Routing a product here (with **Available** on) is also what **publishes it to the customer online-ordering menu** — the public `/order/$token` storefront lists exactly these curated items (`restaurant_menu_items`), **not** the whole catalog, so add every online-orderable item here.

**Recipes** — for a **menu product**, list **ingredient** products with **quantity** + **unit of measure**; feeds theoretical **food cost**.

**Settings** — enable **Public (online) ordering**, toggle **Pickup** and/or **Delivery** fulfilment, and publish the customer ordering **link / QR** (`/order/$token`). Save to apply the fulfilment toggles. See *Online ordering (customer)* below.

### Restaurant POS — `/admin/restaurant-pos`
- **Purpose:** Dine-in / takeaway / delivery ordering against floors, tables and table sessions; sends items to the kitchen (KOT) and settles like the retail POS. Requires the **Restaurant POS** module.
- **Menu source:** the **product catalog** — every active product is a menu tile (searchable, filtered by category); tiles can carry **modifier groups** (see *Menu & Modifiers*).
- **Flow (dine-in):** tap a table → **Open table** (Guests, optional Waiter) → a running order (`RO-…`) opens with a live timer → tap tiles to add lines (each tap adds a line; qty ±, **Remove** before sending) → **Send KOT** fires it to the kitchen (lines flip to *sent*; only **Void**/**Comp** afterwards) → **Request bill** / **Settle**. **Merge** / **Split** combine or divide orders; **Recover** restores a cleared cart; **Cancel** voids the session.
- **Tax** applies from the shop's active tax rate (e.g. 17% VAT) on the running order and the settle total.
- **Settle** — pick a **payment method** and confirm the tendered amount → posts the sale exactly like the retail POS: a balanced journal (**Dr Cash/Bank · Cr Sales Revenue · Cr Tax Payable**, plus **Dr COGS · Cr Inventory**), decrements stock, and frees the table back to *available*.

### Kitchen Display (KDS) — `/admin/restaurant/kds`
- **Purpose:** Live kitchen queue fed by the KOT stream; screens refresh automatically (real-time). Requires the **Restaurant Kitchen** module.
- **Board:** each ticket shows its order (`RO-…`), table, items and age, grouped by station (**All stations** or per station), with header counters **Active / Rush / Late / Served / Bumped**; **Active / Board / History** views plus **Compact** and **Fullscreen** modes.
- **Actions:** **Bump** (mark done — moves the ticket to the *Bumped — recall* strip where **Recall** returns it; the order/payment state is untouched), **Rush** (priority) and **Served** (delivered to table).

### Front-of-house & operations sub-pages
- **Reservations** — `/admin/restaurant/reservations` (bookings: guest, phone, party size, date & time, duration, optional table, special requests). Statuses **pending → confirmed → seated → completed / cancelled / no-show**; **Seat** picks a table + optional waiter and **opens a dine-in order** through the restaurant engine (the table flips to *occupied*).
- **Waitlist** — `/admin/restaurant/waitlist` (walk-in queue: guest, phone, party size, quoted wait). **Notify** tells the guest their table is ready (waiting → notified); **Seat** opens a dine-in order on a chosen table; **Remove** drops them.
- **Delivery** — `/admin/restaurant/delivery`: **New delivery** captures the customer (name, phone, address, delivery fee, notes) and opens a restaurant order — add items in Restaurant POS, **Send KOT** and **Settle** (the same order→KOT→settlement engine as dine-in). Track the lifecycle **new → accepted → preparing → ready → out for delivery → delivered** with the per-order advance button. Operational last-mile assignment / dispatch / ETA lives on **`/admin/restaurant/fulfillment`** (Active / Exceptions / History / External Orders) and never changes the order, payment, settlement or inventory.
- **Online ordering (customer)** — `/order/$token`: the branded self-service storefront customers reach from the **Public Ordering** link / QR (enable it under *Settings*, above). It renders a themed hero with the enabled **Pickup / Delivery** modes, **menu categories**, **featured** + **full menu** (with search + category filter), and the shop's contact details + opening hours. **The menu lists only the items you curated in *Kitchen Setup* (product routing / availability → `restaurant_menu_items`), not the full catalog** — if it shows “No items found”, no items have been curated yet. Customers add items to a cart, choose **Pickup** or **Delivery** (name / phone / address), and place the order, which lands as a restaurant order (`RO-…`) in the same order → KOT → settle engine. New online orders surface in **Restaurant POS → Takeaway orders** (status *received*) to accept → send KOT → settle, with tax applied to the order total. Tiles show **“Only N left”** / **“Unavailable”** from server-authoritative stock.
- **Refunds** — `/admin/restaurant/refunds`: refund a **settled** restaurant order — pick per-line **quantities** to refund (full or partial), a **refund method** and a reason. Reuses the shared return/refund engine (**reversing journal + loyalty reversal**), with **no restock of prepared food** by default; the server re-validates refundable quantities to prevent over-refund and produces a `RET-…` return.
- **Wastage** — `/admin/restaurant/wastage` (record why restaurant stock was lost). Two-step: **Record Waste** (search item, qty in stock unit, stock location, reason SPOILAGE / EXPIRED / DAMAGED / …, notes) creates a draft with a deterministic cost snapshot but **no stock move**; **post** it from **Open / Failed** to decrement stock through the Inventory writer (it never posts Accounting). Feeds **Food cost**.
- **Food cost** — `/admin/restaurant/food-cost` (recipe-driven cost vs price).
- **Analytics** — `/admin/restaurant/analytics` (restaurant performance).

---

## CRM, Segments & Analytics

One **shared customer identity** across POS, Restaurant, Ecommerce, CRM, Loyalty and Marketing — customers are never duplicated. Module `crm`; **reads** open to SELLING_ROLES, **management** to MANAGERS. (Loyalty is documented above under *Promotions, Loyalty & Marketing*.)

### Customer Directory — `/admin/crm`
- **Purpose:** Server-side searchable directory of every customer with a **360° profile**. Search by name / phone / email / code / company; filter by **status**, **tag**, **segment** and **consent**. Opening a customer shows their contact, tags/segments, order history & spend, loyalty and activity timeline — the same record used by POS, checkout and Loyalty.
- **Access:** SELLING_ROLES read; MANAGERS manage.
- **Editable customer record** (via *Edit* here or on `/admin/customers`; owner / admin / manager):

| UI label | Field key | Type | Notes |
|---|---|---|---|
| Name | `full_name` | text | ≤ 120 chars |
| Phone | `phone` | text | ≤ 40 chars |
| Email | `email` | text | ≤ 160 chars |
| City | `city` | text | ≤ 80 chars |
| Group | `group_id` | select | customer group → group pricing in POS/checkout |
| Tags | `tags[]` | tags | ≤ 30 tags, ≤ 40 chars each |
| Notes | `notes` | textarea | ≤ 1000 chars |

Much of the profile (orders, spend, **segment**, last-order date, loyalty points) is **derived from real orders** — the customer record is created automatically at first purchase (name/phone/email) and enriched here.

### Segments — `/admin/segments`
- **Purpose:** Define customer **segments** (rules over spend / recency / tags) reused by Marketing campaigns and directory filters.

### CRM Analytics — `/admin/crm-analytics`
- **Purpose:** Read-only customer analytics (acquisition, retention, RFM-style views) derived from real orders.

### Growth Dashboard — `/admin/growth`
- **Purpose:** Marketing/growth overview across campaigns, segments and customer growth.

---

## HR & Payroll

Module `employees` for HR (MANAGERS); **Payroll** is finance-gated (`owner` / `admin` / `accountant`).

### Employees — `/admin/employees`
- **Purpose:** Team directory. An **employee is an HR record** and may or may not have an application login. Create with First/Last name (plus optional photo, gender, nationality, and the Employment / Organization / Contact / Sensitive / Payment tabs); code auto-assigns `EMP-####`. The saved profile has tabs **Overview, Employment, Contact, Attendance, Leave, Payroll, Documents, Compliance, Lifecycle, Emergency, History, Payment**.
- **Access:** MANAGERS.

### Departments / Designations — `/admin/departments`, `/admin/designations`
- **Purpose:** HR/org master data referenced by employees.

### Attendance — `/admin/attendance`
- **Purpose:** Daily attendance register; feeds payroll. Calculations are **server-authoritative**.
- **Views:** **Daily** (per-day register, filterable by department / branch) and **Register** (range). Header counters: **Present / Late / Absent / Half day / On leave / Day off / Holiday**.
- **Mark attendance** — *Add an employee to today…* → pick employee → set **Status** (Present / Late / Absent / Half day / On leave / Day off / Holiday), **Check in** & **Check out** times and optional **Notes**. The server computes **Worked** hours (e.g. 09:00 → 17:30 = 8h 30m) and rolls an overnight check-out past midnight to the next day automatically. **Mark** on a row re-marks (correction). Approved leave (below) auto-fills those days as *On leave*.

### Leave — `/admin/leave`
- **Purpose:** Leave types, per-employee balances and requests → approvals. **Approved leave flows into attendance automatically.**
- **Leave Types** tab — **Code**, **Name**, **Default days / year** (a reference figure — it is **not** auto-accrued into balances; grant entitlements on the Balances tab), **Paid leave**, **Requires approval**, **Carry forward allowed**.
- **Requests** tab — **New request**: Employee, Leave type, **Day part** (full / half — half only for single-day requests), **Start / End date** (days counted inclusively), Reason → **Submit** (`pending`) → **Approve** / **Reject**. Status filter: All / Pending / Approved / Rejected / Cancelled; an approved request can still be **Cancel**led.
- **Balances** tab — pick an employee + year: columns **Opening / Accrued / Adjusted / Used / Remaining**, where **Remaining = opening + accrued + adjusted − used** and **Used is driven by approved leave** (not editable here). **Adjust balance** sets the opening / accrued / adjusted figures per leave type (this is how annual entitlements are granted).

### Shifts — `/admin/shifts`
- **Purpose:** Shift schedules and assignments.

### Recruitment — `/admin/recruitment`
- **Purpose:** Hiring pipeline (candidates through stages).

### Loans — `/admin/loans`
- **Purpose:** Employee loans / advances and repayment schedules.

### Performance — `/admin/performance`
- **Purpose:** Performance reviews.

### Employee Lifecycle — `/admin/hr/lifecycle`
- **Purpose:** Track and action employment-change workflows — **Probation Review, Confirmation, Transfer, Promotion, Salary Revision** and **Contract Renewal**. The dashboard surfaces counters (probation ending ≤30 days; reviews / confirmations / transfers / promotions / salary-revisions pending; contract renewals due; awaiting application) and a list of actions filterable by **type** and **status** (`draft → submitted → approved → rejected → completed / cancelled`). Individual actions are raised per employee (from the profile's **Lifecycle** tab → *New lifecycle action*: pick **Type**, **Effective date** and **Reason**, then the action's **target chosen by name from a dropdown** — Promotion → **New designation**, Transfer → **Target department / branch**, Salary Revision → **New amount** — never a raw id/uuid box) → **Create draft**. Submitting, approving and **completing** one then applies the change through the authoritative HR path (updating the employee's designation/department/salary), and every step lands on the employee's **Unified timeline**.

### Workforce Requests — `/admin/hr/workforce-requests`
- **Purpose:** A manager **approval inbox** for employee-raised **attendance corrections** and **loan / advance** requests, plus a leave **Calendar** and leave **Carry Forward**. **Approve** authorizes a request and **Apply** routes it through the existing attendance / loan writers; **managers see only their direct reports**. **Carry Forward** rolls eligible remaining leave balance from a source year into a target year (types with carry-forward enabled) — **idempotent** (re-running never double-counts), with a **Preview** before you **Run**.

### Payroll — `/admin/payroll`
- **Purpose:** Calculate, review, approve, **post to the ledger** and pay salaries. Tabs: **Payroll Runs**, **Salary Structures**, **Pay Components**, **Settings**.
- **Access:** `owner` / `admin` / `accountant`.

**1. Settings (one-time, required before posting)** — map the ledger accounts payroll uses: **Salary / earnings expense account** (fallback, e.g. `6100 Salaries`), **Employee payable account** (**required to post/pay** — a liability, e.g. `2000 Accounts Payable`), optional **Deduction liability account**, plus default working days/month, hours/day and overtime multiplier. Pay components can override the expense account per component.

**2. Salary structure** (Salary Structures tab) — pick an employee → **New structure**: **Effective from** (structures are dated; a new one closes the previous period and preserves history), **Base salary**, optional Currency, **Std days / month** (26), **Std hours / day** (8), and optional **allowance / deduction components** (drawn from *Pay Components*). Base-salary-only is valid.

**3. Period → run → calculate → approve → post → pay**

| Step | What happens |
|---|---|
| **New period** | e.g. "September 2026" with start / end / payment dates → status `open` |
| **New run** | names a run for the period (optionally scoped by department / branch) → `draft` |
| **Calculate** | builds each employee's payslip from their structure & attendance; **prorates** for partial-period employment (a mid-period joiner is paid only for days employed) and raises **warnings** (e.g. no shift assignment, proration) → `calculated` |
| **Approve** | confirms the run (if any employee has warnings, a *"Approve anyway?"* prompt appears) → `approved` |
| **Post to accounting** | posts **one balanced journal** (Dr salary/earnings expense · Cr employee payable + deduction liabilities), period-guarded → `posted`. *Editing after calculate requires **Reopen → Recalculate** so the lines re-snapshot the current account mappings.* |
| **Pay** | per employee (Details → *Salary payment*): record the net against **Cash** or **Bank** → payment status `paid` (Dr employee payable · Cr Cash/Bank). A default **Cash Drawer** and **Bank Account** are provisioned automatically (linked to Chart-of-Accounts Cash `1000` / Bank `1100`), so cash payouts work out of the box; older shops are back-filled on their first payout |

**Employee Details** shows attendance stats, the component breakdown, any warnings, a **View payslip** (PDF — never license-gated) button, and an **Add manual adjustment** (earning/deduction) control.
- **Related:** `/admin/hr/payroll-ops` (payroll reports & **reversal**).

---

## Business Intelligence

### Executive Dashboard — `/admin/bi`
- **Purpose:** A unified, **read-only** executive view over existing GoRetailOS reporting. **Figures are UTC-bucketed** (operational sales and Accounting P&L are separate lenses). Pick a **From / To** range and a **Compare** basis (previous period / yesterday / same weekday last week / previous week / previous month / same month last year / none).
- **Access:** role-gated (managers / finance).

### Reports Center — `/admin/bi/reports`
- **Purpose:** A **registry** of read-only BI reports; open any report over a date range and **Export** (server-built CSV). Each report is visible only when its module and your role allow it. **Backend:** `bi-report-registry.ts` (`BI_REPORTS`) → `bi-reports.server.ts`.
- **Reports (by category):**
  - *Sales performance* — Sales Overview, Sales by Product, Sales by Staff, Consolidated Sales
  - *Tax* — Sales Tax; *Returns* — Sales Returns
  - *Valuation* — Inventory Valuation, Inventory by Location; *Stock health* — Low Stock, Dead Stock; *Movements* — Stock Movements, Stock Adjustments
  - *Financial statements* — Profit & Loss, Balance Sheet, Cash Flow, Trial Balance; *Receivables & Payables* — AR Ageing, AP Ageing
  - *Purchase orders* — Purchase Order Status, Purchases by Product, Goods Receipts, Outstanding Purchase Orders; *Supplier analysis* — Supplier Spend
  - *Customer analytics* — Customer Analytics; *Menu* — Menu Performance; *Labour cost* — Payroll Cost

---

## Advanced Procurement

A sourcing pipeline **in front of** Purchasing: internal **requisitions** → approval → **RFQ** to suppliers → **quotations** → **comparison** → **award** → optional **contract** → convert to a **purchase order**. Module `procurement`; access via procurement permissions (owner/admin allow-all, otherwise per-role `role_permissions` with built-in defaults). Hub at `/admin/procurement` (requisition / sourcing / governance / vendor-portal KPIs). **Backend:** `procurement.server.ts` (requisitions), `procurement2.server.ts` (RFQ + awards), `procurement3.server.ts` (contracts).

### Requisition — `/admin/procurement/requisitions`
- **Purpose:** Raise an internal purchase request; **submit** for approval, then it is **decided** (approved / rejected) per the approval policy. Numbered `REQ-####`.

| UI label | Field key | Req | Notes |
|---|---|---|---|
| Branch | `branch_id` | No | scopes the request |
| Department | `department_id` | No | cost owner |
| Required date | `required_date` | No | when it's needed |
| Priority | `priority` | No | low / normal / high / urgent |
| Justification | `justification` | No | why it's needed |
| Budget reference | `budget_reference` | No | links to a budget line |
| Notes | `notes` | No | — |
| Items | `items[]` | Yes | each: variant/description, quantity, unit, estimated price |

### RFQ — `/admin/procurement/rfqs`
- **Purpose:** Invite selected suppliers to quote (optionally **created from a requisition**). Numbered `RFQ-####`.

| UI label | Field key | Req | Notes |
|---|---|---|---|
| Title | `title` | Yes | — |
| Description | `description` | No | — |
| Issue date / Response deadline | `issue_date` / `response_deadline` | No | deadline drives "awaiting responses" |
| Expected delivery / location | `expected_delivery_date` / `delivery_location` | No | — |
| Payment terms / Terms | `payment_terms` / `terms` | No | — |
| Suppliers | `supplier_ids[]` | No | who to invite (also via the vendor portal) |
| Items | `items[]` | Yes | lines to be quoted |

### Quotations & comparison — `/admin/procurement/quotations`, `/admin/procurement/comparisons`
- **Purpose:** Record or receive supplier **quotations** against an RFQ (vendor-portal submissions land here), then **compare** them side-by-side to pick the best.

### Award — `/admin/procurement/awards`
- **Purpose:** Award the RFQ to a supplier/quote; add charges to reach the award total, then **convert to a PO**.

| UI label | Field key | Req | Notes |
|---|---|---|---|
| Supplier | `supplier_id` | Yes | winning supplier |
| RFQ / Quote | `rfq_id` / `quote_id` | No | source |
| Payment / Delivery terms | `payment_terms` / `delivery_terms` | No | — |
| Freight / Other / Tax / Discount | `freight_amount` / `other_charges` / `tax_amount` / `discount_amount` | No | roll into the award total |
| Items | `items[]` | Yes | awarded lines |

### Contract — `/admin/procurement/contracts`
- **Purpose:** A governing agreement with a supplier (often **from an award**); tracks value, dates and expiry. **Expiry status** derives from `end_date` (active → **expiring** within 60 days → **expired**).

| UI label | Field key | Req | Notes |
|---|---|---|---|
| Supplier / Title | `supplier_id` / `title` | Yes | — |
| Contract type | `contract_type` | No | — |
| Start / End / Renewal date | `start_date` / `end_date` / `renewal_date` | No | `end_date` drives expiry status |
| Contract value | `contract_value` | No | — |
| Owner | `owner_user_id` | No | responsible person |
| Items | `items[]` | No | each: description, agreed price, unit, quantity ceiling, minimum commitment, discount, valid-until |

### Approval inbox / Suppliers / Reports / Settings
- **Approval inbox** — `/admin/procurement/approvals` (pending requisition/award approvals for the signed-in approver).
- **Suppliers** — `/admin/procurement/suppliers` (supplier records with a vendor-portal invite).
- **Reports** — `/admin/procurement/reports` (sourcing analytics).
- **Settings** — `/admin/procurement/settings`: approval policy — **require approval**, **approval threshold**, **approver roles**, **requisition prefix**.

### Vendor portal (supplier-facing) — `/store/$shopSlug/vendor` + `/vendor.invite.$token` + `/vendor.rfq`
- **Purpose:** Invited suppliers open their RFQ via a tokened link and submit a **quotation** that lands in *Quotations* — no admin login required.

---

## Appendix A — Enumerations

Many forms share these fixed choice lists (defined as Postgres enum types). Where a form has a "status" or "type" select, it almost always draws from one of these:

| Enum | Used by | Values |
| --- | --- | --- |
| `shop_role` | Team & Roles, invitations | owner · admin · manager · cashier · accountant · inventory · staff |
| `product_status` | Product | draft · active · archived |
| `order_status` | Orders | pending · confirmed · processing · completed · cancelled · refunded |
| `order_channel` | Orders | pos · online · ai |
| `payment_status` | Orders, invoices | unpaid · partial · paid · refunded |
| `fulfillment_status` | Orders | unfulfilled · processing · shipped · delivered · picked_up · returned |
| `movement_type` | Inventory movements | opening · purchase · sale · return · adjustment · transfer_in · transfer_out · count |
| `transfer_status` | Transfers | draft · requested · approved · shipped · received · cancelled |
| `po_status` | Purchase orders | draft · submitted · approved · partial · received · cancelled |
| `account_type` | Chart of accounts | asset · liability · equity · income · expense |
| `approval_status` | AI action approvals | pending · approved · rejected · executed |
| `ai_agent` | AI | salesman (storefront) · assistant (admin) |

---

## Appendix B — Tables ↔ forms index

The database has ~75 tables, all scoped by `shop_id`. Key foreign keys (the wiring behind the "Connections" blocks):

| Pointer field | Points at | Meaning |
| --- | --- | --- |
| `shop_id` | shops | tenant scope on every table |
| `branch_id` | branches | branch binding |
| `category_id` | categories / expense_categories | product category · expense category |
| `parent_id` | categories / accounts / storefront_navigation | tree self-reference |
| `brand_id` | brands | product brand |
| `tax_rate_id` | tax_rates | product/line tax |
| `product_id` | products | variant/image parent |
| `variant_id` | product_variants | the sellable unit on order/PO/cart/count/transfer/return lines |
| `customer_id` | customers | order/cart owner |
| `order_id` / `order_item_id` | orders / order_items | sale + returned line |
| `return_id` | returns | refund parent |
| `register_id` | registers | POS till |
| `cash_session_id` | cash_sessions | open till session |
| `location_id` / `from_location_id` / `to_location_id` | inventory_locations | stock location(s) |
| `transfer_id` | inventory_transfers | transfer lines |
| `stock_count_id` | stock_counts | count lines |
| `supplier_id` | suppliers | PO / invoice / payment |
| `purchase_order_id` | purchase_orders | receipt/invoice source |
| `goods_receipt_id` | goods_receipts | receipt lines |
| `supplier_invoice_id` | supplier_invoices | supplier payment target |
| `account_id` | accounts | journal line / expense account |
| `journal_entry_id` | journal_entries | ledger lines |
| `document_id` | ai_knowledge_documents | knowledge chunks |
| `conversation_id` | ai_conversations | AI messages |

*Per-domain table ownership is listed at the end of each section above.*

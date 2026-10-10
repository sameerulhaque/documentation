# GoRetailOS — Detailed Product Features (per module: fields, flows & connections)

> **The complete per-module feature compendium** — every module's entities with their real fields
> (extracted from the server input validators on 2026-10-03), the lifecycle flows, what each action
> updates elsewhere, and the routes. Companion docs: `SUMMARY.md` (positioning),
> `GORETAILOS_WEBSITE_CONTENT_AND_POSITIONING.md` (marketing voice),
> `../manuals/USER_MANUAL.md` (click-level form tables — its section anchors are referenced below),
> `features.md` (engineering register). §16 lists **current boundaries** so nobody oversells.

**Notation:** `(req)`/`(opt)` from the validators; `enum(a|b)` = allowed values; `≤N` = max length;
`shopId` accompanies every call and is omitted. Roles: owner, admin, manager, cashier, accountant,
inventory, staff — server sets: MANAGERS (owner/admin/manager), ADMINS, INVENTORY_ROLES (+inventory),
FINANCE_ROLES (+accountant), SELLING_ROLES (+cashier), PAYROLL_ROLES (owner/admin/accountant).

## 0. Platform-wide conventions

- **Licensing:** every module function file guards role **and** module license
  (`moduleGuards(key)`); shared surfaces use `"ANY"` (trial or any license). The offline POS sync
  queue is deliberately license-exempt (business continuity).
- **Seeded chart of accounts** (auto-postings look up by code): 1000 Cash · 1100 Bank · 1150 AR ·
  1200 Inventory · 2000 AP · 2100 Tax Payable · 3000 Owner Equity · 4000 Sales Revenue · 4100 Other
  Income · 5000 COGS · 6000 Opex · 6100 Salaries · 6200 Rent · 6300 Utilities.
- **Period guard:** GL writes into a closed/locked fiscal period are blocked everywhere (POS,
  checkout, restaurant, purchasing, vouchers, payroll, loans, counts, imports, AI actions).
- **Journal reference types in use:** order/sale, return, expense, supplier_invoice,
  supplier_payment, opening_stock, stock_count, voucher, voucher_reversal, opening_balance,
  payroll_run, payroll_payment, employee_loan, loan_repayment.
- **Shop provisioning seeds:** owner membership, Main Branch, Register 1, Main Store location,
  payment methods (cash, card, bank, credit [disabled], cod), Standard VAT 0%, the chart of
  accounts, document sequences (ORD- RCP- INV- PO- RET- CN- DN- GRN- PRD-), AI permissions,
  storefront defaults, expense categories, cash & bank accounts, and the **90-day TRIAL
  subscription**.

---

## 1. Retail POS (`retail_pos`) — manual §Sales

**Entities & fields**
- **POS sale** `posCheckout`: items[≥1]{variant_id (req), quantity 0.001–9999 (req), discount 0–1M (opt)}; payments[]{method_code ≤40 (req), amount ≥0 (req), reference ≤120 (opt)}; customer_id, register_id, branch_id (opt); notes ≤500; offline identity local_uuid ≤80 / device_id ≤80 / local_seq / offline_created_at / offline_code ≤60 (opt); client_snapshot{lines, subtotal, discount_total, tax_total, total} (opt); credit bool, override_credit_limit bool (opt).
- **Offline sync batch** `syncOfflineSales`: up to 50 sales of the same schema (credit forced off).
- **Return/refund** `posRefund` (MANAGERS): orderId; items[≥1]{order_item_id, quantity ≥0.001, restock (def true)}; reason ≤300; method_code (def cash; UI cash|card|bank).
- **Credit settlement** `recordSalePayment` (managers+accountant): orderId; amount >0 (≤ outstanding); method_code (def cash); reference ≤200.
- **Delete sale** `deleteSaleFn` (ADMINS): hard delete with full reversal.
- **Register** (MANAGERS): name 1–120 (req), code ≤40, branch_id, is_active; user assignment registerId+userId.
- **Cash session:** open{registerId, openingBalance ≥0}; close{sessionId, countedBalance ≥0}; movement{sessionId, movementType enum(cash_in|cash_out|expense), amount ≥0.01, note ≤300, categoryId (opt)}.
- **Sync conflict** (MANAGERS): conflictId; action enum(accept|reject|retry); note ≤500. Kinds: price_variance|tax_variance|rejected; statuses pending|retrying|resolved|rejected.
- UI facts: tenders cash|card|bank|credit|other; shortcuts F2 search · F4 customer · F6 discount · F8 hold · F9 pay.

**Flows**
- Shift: open session (`opening` movement) → sales → cash movements → close (expected = opening + cash sales − cash refunds + in − out − expenses; discrepancy = counted − expected) → `closing` movement.
- Sale → order completed/picked_up; payment_status paid | partial | unpaid (credit). Credit needs `customers.credit_allowed` + role in credit_roles + a customer; limit checked (manager override). Later payments move partial → paid.
- Return: full → refunded; partial → partial. Unpaid orders can't be refunded.
- Offline: client queue → sync (dedup by local_uuid). Price/tax drift posts at the charged price + a variance conflict; failures store a rejected conflict for retry.

**Cross-impacts**
- Writes orders, order_items, payments, receipts; `sale` stock movement per line (negative stock blocked unless allowed/oversell role; offline replays exempt).
- GL: Dr Cash/Bank/AR · Cr Revenue 4000 + Tax 2100 · Dr COGS 5000 / Cr Inventory 1200 at avg cost. Refund reverses (+restock reversal).
- Also: cash movement into the open session, low-stock alerts, `SaleCompleted` event, loyalty earn; cash `expense` movements create approved expenses + GL; closing discrepancy >5% raises a critical alert.
- Server enforces `pos.discount_roles` and per-role `pos.payment_methods`.

**Routes:** /admin/pos · /admin/registers · /admin/sales · /admin/sync · /admin/orders (returns) · /admin/sales-reports · /offline-pos · /pos-display (customer screen via BroadcastChannel).

## 2. Restaurant POS (`restaurant_pos`) — manual §Restaurant

**Entities & fields**
- **Open order:** dine-in{table_id (req), guest_count 1–100, waiter_employee_id, notes ≤300}; takeaway{waiter, customer_name ≤160, customer_phone ≤60, notes}; delivery{customer fields, delivery_fee (def from settings), delivery_notes, source}.
- **Item** `addRestaurantItem`: orderId, product_id (req); variant_id; quantity 0.001–100000 (def 1); special_instructions ≤200/300; modifiers[≤30]{group, option}. Plus updateQuantity / remove.
- **Manager actions:** void item{quantity, reason}; set complimentary; order charges{discount_total, service_charge_percent 0–100}; transfer table; cancel{reason}; merge/unmerge table; merge occupied orders; reopen; recover settlement.
- **KOT:** send(orderId); setKotStatus enum(sent|preparing|ready|completed|cancelled).
- **Settle:** payments[1–10]{method_code ≤30, amount, reference}; register_id (opt).
- **Splits:** by amount{≥0.01} or items{allocations[1–200]}; pay split (payments[1–10]); void split.
- **Refund** (manager): items[1–200]{order_item_id, variant_id?, quantity, unit_price, total, restock (def false)}; method_code; reason.
- **Menu config:** modifier group{name, selection_type single|multiple, min/max selections, required, sort, active}; option{group, name, price_adjustment ±1M, sort, active}; product↔group assignment; menu item{product, kitchen_station_id, is_available, prep_time 0–600, sort}; recipe{product/variant, yield 0–100000, items[≤100]{ingredient variant/product, quantity, uom ≤20, notes}}.
- **Settings:** default_guest_count, default_service_charge_percent, allow_table/order_transfer, require_waiter, allow_item_notes, auto_send_kot, menu_availability_mode enum(manual|recipe_inventory|recipe_inventory_with_override), low_stock_threshold, public_ordering_enabled, pickup_enabled, delivery_enabled.
- **Public ordering** (anonymous, by rotating token): fulfillment pickup|delivery; items[1–100]; customer{name, phone (req), email}; delivery{address (req), notes}; client_uuid.
- **Delivery:** status enum(new|accepted|preparing|ready|out_for_delivery|delivered|cancelled); fulfillment overlay{type IN_HOUSE|EXTERNAL_AGGREGATOR|EXTERNAL_COURIER; assign/unassign employee; ETA ≤40; advance PENDING→ASSIGNED→READY_FOR_PICKUP→DISPATCHED→PICKED_UP→DELIVERED; FAILED/CANCELLED terminal}; aggregator import{provider, external ids, free-text items[]} (idempotent).
- **Restaurant website** (site_key "restaurant"): settings{isPublished, primaryColor/accentColor (#rrggbb), announcement ≤200, hero{headline ≤200, subheadline ≤400, image_url, cta}, socialLinks, seo{title ≤200, description ≤300}}; banners; nav links{header|footer, label ≤60, url, position, visible}; pages{slug, title ≤160, content ≤20000, isPublished}.

**Flows**
- Order: open → sent (first KOT) → billing (bill requested or split created; edits lock) → partially_paid → settled; cancel only before any payment; reopen voids open splits. Line: pending → sent (sent_quantity), with voided_quantity / complimentary. Split: open → paid|void; last paid split settles the order. Table session: open → billing → closed|cancelled.
- Public/QR orders land as restaurant orders (channel online/qr), auto-KOT if configured.

**Cross-impacts**
- Pricing via priceCart + modifier surcharges; recipe modes gate un-preparable items; `sendKot` best-effort reserves ingredients.
- Settle = ONE sale via record_pos_sale (idempotent `resto-<id>`): service-charge/discount lines; recipe-expanded ingredient consumption (or the product itself, not enforced); GL as POS; session cash movement; merged tables released; loyalty earn. Cancel releases reservations, writes nothing financial. Refund via shared return (no restock by default).

**Routes:** /admin/restaurant (overview) · /admin/restaurant-pos · /admin/restaurant/{menu,recipes,settings} · /admin/restaurant/{delivery,fulfillment,refunds,food-cost,analytics} · /admin/restaurant-site. Public: /order/$token · /store/$slug/dine (+ /p/$pageSlug).

## 3. Restaurant Tables (`restaurant_tables`)

**Entities & fields**
- Floor{name 1–120 (req), branch_id, description ≤500, sort, active}; Table{code 1–40 (req), name ≤120, branch/floor, capacity 1–100, position_x/y, width/height, shape ≤20, active}; floor layout save (manager): items[≤500]{tableId, pos, size, shape, rotation, z_index}.
- Table session: open{table_id, guest_count, waiter, notes} / updateGuests / assignWaiter / close / cancel.
- Service state: NORMAL|CLEANING|BLOCKED|HELD {reason ≤300, expiresAt}; clear.
- Table QR (manager): rotate{tableId, enabled} → qr_token/qr_enabled. QR self-order (anonymous): items[1–100]{product, variant?, quantity 1–50, modifiers, special_instructions ≤200}, guest_name ≤120, guest_phone ≤60.
- Reservation: branch, customer/guest fields, party_size, reserved_at, duration (def 90), table_id, notes, special_requests, source; status enum(pending|confirmed|seated|completed|cancelled|no_show); seat{tableId?, waiterId?}.
- Waitlist: guest fields, party_size, preferred_floor, quoted_minutes; status enum(waiting|notified|seated|cancelled|no_show); seat{tableId (req)}.

**Flows:** reservation pending → confirmed (overlap + capacity checked) → seated (opens a dine-in order, source "reservation") → completed; waitlist waiting → notified → seated (source "waitlist"). Floor status precedence: OCCUPIED/BILLING > BLOCKED/CLEANING/HELD > RESERVED_SOON > AVAILABLE. QR: /r/$token → context (qr_ordering_enabled) → menu → submit (attaches to open order or opens dine-in) → status received/preparing/ready/served.

**Cross-impacts:** seating creates session+order; layouts/service states are overlays that never touch orders.
**Routes:** /admin/restaurant/{tables,floor-plan,reservations,waitlist}. Public: /r/$token.

## 4. Restaurant Kitchen (`restaurant_kitchen`)

**Entities & fields**
- Station{code 1–40, name 1–120 (req), branch, sort, active}. KDS status: new|preparing|ready|completed|cancelled. Overlay events: bump/recall, rush priority, served mark, course state{starter|main|dessert|drinks|other, fired}, item-course assignment.
- **Wastage** (INVENTORY_ROLES): location_id (req); reason_code enum(SPOILAGE|EXPIRED|PREPARATION_WASTE|OVERPRODUCTION|DAMAGED|SPILLAGE|QUALITY_REJECT|STAFF_MEAL|OTHER) (req); branch, notes ≤1000, occurred_at; items[1–50]{variant_id, quantity >0, uom ≤40, recipe_context ≤300}. Actions post/retry/cancel.

**Flows:** KDS new → preparing → ready → completed (ready→preparing allowed; cancel from any open state); timestamps started/ready/completed; two screens can't double-advance. Wastage: RECORDED (items PENDING) → post (per line POSTING → POSTED|FAILED) → POSTED/PARTIALLY_POSTED/FAILED; retry per line; cancel only before any line posts.
**Cross-impacts:** wastage posts `adjustment` movements with cost snapshot, **no GL**; station routing on menu items decides ticket targets.
**Routes:** /admin/restaurant/kds · /admin/restaurant/kitchen · /admin/restaurant/wastage.

## 5. E-commerce (`ecommerce`) — manual §Storefront (Editor + Customer forms)

**Entities & fields**
- **Online order** `placeOnlineOrder` (public; 10/min/IP): items[≥1]{variant_id, quantity 0.001–9999}; customer{full_name 1–120 (req), email (req), phone ≤40, user_id}; shipping_address{line1, city, country, postal_code}; payment_method_code (def cod; UI cod|bank); notes ≤500; client_uuid; selected_gift_variant_ids[≤20].
- **Cart quote:** items; destination{city,country}; gift selections — server prices everything incl. delivery fee (zones/slabs) and free-gift entitlement.
- **Payment proof** (public): orderNumber; contactEmail; file jpg/png/webp/pdf ≤5 MB. Review: verify|reject{reason ≤300}.
- **Track order:** orderNumber + email-or-phone. **Customer profile:** full_name/email/phone. **Addresses:** label, line1/2 ≤200, city/state/postal/country/phone, is_default (+delete/set-default).
- **Review** (public, 6/min/IP; verified purchase): productId, orderNumber, rating 1–5, title ≤120, body ≤2000, authorName ≤80; moderation publish|reject.
- **Storefront editor:** theme draft{homepage_variant, colors (primary/secondary/accent/text/background), font_family, announcement_bar, header, footer, hero, social_links, seo, theme_code}; banners{placement, title, subtitle, image, link, position, visible}; sections{page_slug, section_type, title, config, position, visible}; nav items{location, label, url, parent, position, visible}; policies{privacy|terms|return|shipping|faq, title, content, published}; pages{slug, title, content, published}; contact/branding{phone, email, whatsapp, address, logo, favicon, tagline, name}.
- **Domains** (ADMINS): add hostname 3–253; verify; set primary; remove; **Serves** site = restaurant|null.
- **Fulfillment label** (MANAGERS): unfulfilled|processing|shipped|delivered|picked_up|returned.
- **Funnel events:** ProductViewed|ProductSearched|CategoryViewed|AddedToCart|RemovedFromCart|CheckoutStarted.

**Flows**
- Quote → place (channel online, confirmed, unpaid, unfulfilled) → bank: proof submitted → verified (paid) | rejected; COD collected later via recordSalePayment. Fulfillment unfulfilled → processing → shipped → delivered (or picked_up/returned). Returns via posRefund.
- Editor drafts (is_draft) → member preview → publish | discard. Reviews pending → published|rejected. Domains pending → verified → primary; a Serves-assigned domain serves the restaurant site at root.

**Cross-impacts:** order reuses/creates the customer; record_pos_sale writes order/items/receipt/stock; GL Dr AR (cod/bank) or Bank · Cr Revenue (incl. delivery fee) + Tax · COGS/Inventory; free-gift lines at zero price (stock+COGS still post); low-stock + new_order alerts; `OrderCreated` event. Proof verify posts Dr Bank / Cr AR. Lapsed ecommerce license (enforcement on) refuses new orders.

**Routes:** /admin/storefront (Theme/Banners/Sections/Navigation/Pages/Policies/Contact/Domains) · /admin/domain-setup · /admin/reviews · /admin/orders. Storefront: /store/$slug {index, shop, category/$slug, product/$slug, cart, checkout, track-order, account, sign-in/up, forgot-password, wishlist, compare, contact, about, page/$slug, flash-deals, payment.success/cancel}.

## 6. Inventory (`inventory`) — manual §Catalog + §Inventory + §Manufacturing

**Entities & fields**
- **Product** (MANAGERS): name 1–200 (req), display_name ≤200, product_code ≤60 (auto PRD-######), category_id, brand_id, status enum(active|draft) (req), short_description ≤500, description ≤20000, unit ≤24 (def pcs), tags[≤30], tax_rate_id, flags is_featured/new_arrival/best_seller/is_online (def true), variants[≥1] (req), images[≤30]{url (req), alt ≤200, variant_id}.
- **Variant:** sku 1–80 (req, unique), name ≤120, attributes{}, cost 0–10M (req), price 0–10M (req), sale_price, weight, is_active (def true), tax_rate_id, opening_stock, barcode ≤64.
- Category{name (req), slug, parent, description, image, tax_rate, position, active, online}; Brand{name, slug, logo, description, active, online}; Attribute{name ≤80 + values}; Tax rate (MANAGERS){name, rate, inclusive, default (single), active}; Location{name, branch, default (single), active}.
- **Barcodes:** generate{variantId, ean13|code128}; attach{code 1–64}; deactivate; bulk generate; labels printed{entries[{variantId, count}]}.
- **Adjustment:** variant_id, signed quantity, note → movement + balance (default location).
- **Transfer:** from/to location (req), notes ≤500, items[≥1]{variant, quantity 0.001–999999, unit_cost?}; transitions requested|approved|shipped|received|cancelled with receivedQuantities.
- **Stock count:** location (req), note, scope variant_ids/category_ids/all; progress items{counted_qty ≥0}; approve (MANAGERS).
- **BOM:** finished_variant (req), name 1–160 (req), version, output_quantity >0 (req), active, effective_from, notes ≤2000; components[{variant, quantity, uom ≤20, type material|packaging, wastage_pct, sort}]; extra costs[{type LABOR|OVERHEAD|OUTSOURCED_SERVICE|OTHER, label ≤120, amount ≥0, sort}]; production plan (read-only: bom, desiredQuantity, location).
- **Import** (1–5000 rows): products{product_name, sku, price (req) + code/cost/sale_price/opening_stock/barcode/category/brand/variant/attributes}; suppliers{name + contact/terms}; inventory{sku|barcode, quantity (req), branch, cost}.

**Flows:** transfer draft → requested → approved → shipped (available = qty − reserved) → received (per-line); cancel before shipping. Count draft (snapshot) → counting → completed. Product save handles price history, costs, sale prices, barcodes, opening stock.

**Cross-impacts:** price change → price_history + `PriceChanged`; new-variant opening stock → `opening` movement + GL Dr Inventory/Cr Equity; changed opening on existing → adjustment (no GL). Manual adjustment: movement only. Transfers: transfer_out/in, no GL. Count approve: `count` movements + GL (overage Dr 1200/Cr 5000; shrinkage reverse) at avg cost.

**Routes:** /admin/products · categories · brands · attributes · labels · import · inventory · locations · transfers · stock-counts · manufacturing · inventory-reports (valuation, by-location, low/dead stock, turnover, movements, adjustments) · tax.

## 7. Purchasing (`purchasing`) — manual §Purchasing & Accounting

**Entities & fields**
- **Supplier** (managers+inventory+accountant): name (req), code, email, phone, address, tax_number, payment_terms (cod|net_7|net_15|net_30|net_60), credit_limit, is_active.
- **Purchase order:** supplier + lines{variant, quantity, unit_cost, total} → PO-number, status submitted; editable until received; cancel; ADMIN hard-delete with full reversal.
- **Goods receipt (GRN):** per-line quantities clamped to outstanding → GRN-number at the default location.
- **Supplier invoice** (FINANCE_ROLES): supplier (req), invoice_number 1–80 (req, unique), dates, PO/GRN link, total ≥0 (req), notes ≤2000; create-from-source; post / cancel / refresh-overdue.
- **Supplier payment:** invoice (req), amount >0, method_code (req), reference ≤200, paid_at.

**Flows:** PO submitted → partial → received | cancelled (award-created POs start draft). GRNs can split. Invoice draft → posted → partially_paid → paid (overdue derived); cancel while unpaid.

**Cross-impacts:** GRN posts `purchase` movements + cost update (last cost; weighted avg) — **no GL at receipt**; invoice post Dr Inventory(1200)/Cr AP(2000); payment Dr AP / Cr Bank|Cash. PO delete reverses bills/payments/journals/receipts/stock (avg cost not recomputed).

**Routes:** /admin/purchasing · suppliers · purchasing-reports (PO status, supplier spend, by product, receipts, outstanding, AP ageing, cost history) · payables.

## 8. Advanced Procurement (`procurement`) — manual §Advanced Procurement

**Entities & fields**
- Settings{require_approval, approval_threshold 0–1e9, approver_roles[]}.
- **Requisition:** branch, department, required_date, priority low|normal|high|urgent, justification ≤4000, budget_reference ≤120, notes, currency; items[≤200]{variant?, preferred_supplier?, description 1–300 (req), quantity >0, unit ≤40, estimated_price ≥0 (req), notes}. decide approve|reject|return{comments}; comment 1–2000; cancel{reason}.
- **Approval rule:** entity requisition|award, name, min/max amount, branch/department, steps[1–10]{order, role}, active, priority 0–10000.
- **RFQ:** title 1–200 (req), links (requisition), description ≤4000, dates (issue/deadline/delivery), delivery_location ≤300, currency, payment_terms ≤300, terms ≤4000, supplier_ids[≤50], items[≤200]{description (req), quantity >0, unit, target_price, specification ≤2000, required_date}; add/remove supplier; issue; status responses_pending|evaluation|closed|cancelled.
- **Quotation:** rfq+supplier (req), quote_reference ≤120, dates, currency, terms, lead days 0–3650, freight/other/tax/discount amounts, attachment, items{rfq_item?, quantity, unit_price ≥0, discount, tax, delivery_date, warranty ≤300}.
- **Award:** supplier (req) + sources (rfq/requisition/quote), amounts, justification ≤4000, items[1–200]{description, quantity, unit_price}; submit → decide → finalize → convert to PO; cancel.
- **Documents:** per entity (requisition|rfq|quotation|award|supplier|compliance|contract), file base64 → procurement-documents bucket.
- **Supplier management:** profile{qualification pending|approved|preferred|conditional|suspended|disqualified, risk low|medium|high, dates, notes}; categories; compliance types + documents{issue/expiry, status pending|valid|expired|rejected; verify}; evaluations{criteria[≤20]{criterion, rating 0–5}, comments}; contacts{name, email (req), phone, designation, primary, active}.
- **Contract:** supplier (req), title 1–200 (req), type ≤80, scope fields, value, owner, terms ≤8000, items[≤200]{type stock|non_stock|service, agreed_price, quantity_ceiling, minimum_commitment, valid_until}; status under_review|active|terminated|closed.
- **Vendor portal** (HttpOnly cookie session): invitations (create/revoke/regenerate); supplier-side quote draft/acknowledge/decline{reason ≤500}/submit + document upload.

**Flows:** requisition draft → submitted (auto-approve under threshold) → under_review (multi-step) → approved|rejected|returned → converted; RFQ draft → issued (≥1 supplier & item) → responses_pending → evaluation → awarded → closed; quote draft → submitted → awarded; award draft → submitted → under_review → approved → finalized → converted (PO draft; handoff idempotent; requisition → converted); contract draft → under_review → active → terminated|closed; invitation active → viewed → accepted → responded|declined|revoked|expired. Every transition writes status history + approvals + audit.

**Routes:** /admin/procurement{, requisitions(/$id), rfqs(/$id), quotations(/$id), comparisons, awards(/$id), contracts(/$id), suppliers(/$id), approvals, reports, settings}. Vendor: /vendor.invite.$token · /vendor.rfq · /store/$slug/vendor.

## 9. Accounting (`accounting`) — manual §Purchasing & Accounting

**Entities & fields**
- Account (FINANCE_ROLES): code, name, type asset|liability|equity|income|expense, parent, active (system accounts protected). Expense category{name, account, active}. Payment method (MANAGERS){name, code, type, position, enabled}.
- **Expense:** category?, amount >0, note ≤500, payment_method (def cash), branch, attachment ≤500; review approve|reject (threshold-based pending).
- **Voucher:** type enum(journal|payment|receipt|contra|sales|purchase|debit_note|credit_note) (def journal), date (req), reference ≤120, narration ≤1000, party ≤200, branch, notes ≤2000, attachment; lines[2–200]{account (req), debit/credit (def 0), narration ≤300, branch/department/cost_center}.
- **Fiscal year:** name/start/end (req), is_current; periods generate; period status open|closed|locked; opening balance lines[2–500]{account, debit, credit}.
- Department{code 1–40, name 1–160, parent, sort, active}; Cost center{code, name, department, branch, parent, active}.
- **Bank reconciliation:** bank_account (req), statement window (req), opening/closing balances, notes; lines{transaction_date (req), value_date, description ≤500, reference ≤200, debit/credit, running_balance, external_id} (CSV ≤5000); match/unmatch/ignore/autoMatch/confirm.
- **Budget:** name, fiscal_year, version, branch (draft); lines{account|account_code, period_month, department, cost_center, amount}; status draft|approved (ADMINS)|closed; budget-vs-actual.
- **Payment allocation:** party customer|supplier, refs, amount ≥0.01 vs remaining/outstanding.

**Flows:** voucher draft → approved → posted → reversed (only drafts editable). Expense auto-posts or pending → approve|reject. Period open → closed → reopen; → locked (final). Reconciliation draft → in_progress → completed; lines unmatched → suggested → matched|ignored. Budgets draft → approved → closed.

**Cross-impacts:** expense GL Dr category account (else 6000)/Cr Cash|Bank; opening balance journal. Read-only reports: P&L (+statement), balance sheet, cash flow, GL (by account/CC/dept), trial balance, validation, AR/AP ageing + statements + summaries, cash book, dimensional TB, comparative P&L, management dashboard.

**Routes:** /admin/finance · /admin/accounting (reports, vouchers, periods, dimensions, bank rec) · accounts · expense-categories · receivables · payables · budgets · payments.

## 10. Employees / HR (`employees`) — manual §HR & Payroll

**Entities & fields**
- **Employee** (MANAGERS): first/last name 1–120 (req), employee_code ≤40, middle/display name, photo, email/phone/alt-phone, address ≤500, city, country, branch/department/cost_center/designation, manager, employment_type ≤30, joining_date, gender, nationality. Status: active|on_leave|suspended|resigned|terminated|inactive{leaving_date, reason ≤500}.
- Sensitive (ADMINS): national_id/passport ≤60, date_of_birth. Payment details (ADMINS): method ≤40, bank fields, IBAN ≤60. Emergency contacts{name 1–160, relationship ≤80, phones, address ≤300, primary}. Documents{type 1–60, title 1–200, file jpg/png/webp/pdf ≤10 MB, issue/expiry, notes ≤500}. User link/unlink.
- Designation{name 1–160, code ≤40, description ≤1000, department, active}.
- **Attendance:** mark{employee, date, status enum(present|absent|late|half_day|on_leave|day_off|holiday), check_in/out_at, notes ≤500}; checkIn/checkOut; approve.
- **Leave:** type{code 1–40, name, paid, requires_approval, default_days, carry_forward, active}; balance adjust{employee, type, year, opening/accrued/adjusted}; request{employee, type, start/end (req), day_part full|first_half|second_half, reason ≤1000} → approve/reject{notes}/cancel; carry-forward{source/target year, type?, max_days}.
- **Shifts:** shift{code, name, start/end HH:MM, break/grace/min-work minutes, overnight, working_days[0–6], active}; assignment{employee, shift, effective_from (req), effective_to, branch, notes}; holiday{name, date, branch, paid, notes}.
- **Loans:** {employee, kind loan|advance, loan_type, principal >0, interest_total, issue_date, first_due_date, installment_count, frequency (def monthly), notes, branch} → approve (ADMINS) → issue (ADMINS, fundingAccountCode?) → repay{amount ≥0.01, method cash|bank|adjustment, date, account_code, note}.
- **Recruitment:** job opening{title, department, designation, branch, type, positions, hiring manager, description, requirements, dates}; candidate{name, contact, resume, source}; application stage enum(applied|screening|shortlisted|interview|offer|hired|rejected|withdrawn); interviews{interviewer, scheduled_at, type, rating, recommendation}; offer (ADMINS){joining date, salary, type, designation/department/branch} status draft|sent|accepted|declined|expired|cancelled; hire{employee_code?, joining_date?}.
- **Performance:** cycle{name, windows} status draft|active|review|closed; goals{title, target, weight, progress, rating}; reviews{reviewer, weighted overall, manager/employee comments} status draft|submitted|reviewed|acknowledged|closed.
- **Compliance:** document types{code, name, category, requires issue/expiry/verification, reminder days}; requirements (by branch/dept/designation/type); employee compliance{document link, responsible user, notes ≤2000, renewal_due_at}; verify verified|rejected; renewals; reminders generate/complete/dismiss. Derived statuses MISSING → … → RENEWAL_IN_PROGRESS.
- **Lifecycle actions:** type enum(probation_review|confirmation|transfer|promotion|salary_revision|contract_renewal){effective_date, reason ≤2000, payload per type}. **Self-service /admin/my:** profile patch; leave request; attendance correction{type enum(6 kinds), requested times/status, reason}; financial request{loan|advance, amount, installments 1–240}; leave attachments.

**Flows:** attendance events → recompute (late/OT/worked from shift+grace) → approve. Leave pending → approved (consumes balance; marks attendance) | rejected | cancelled (restores). Loans draft → approved → active → settled. Recruitment applied → … → hired (creates employee). Lifecycle draft → submitted → approved → completed (applies transfer/promotion/salary/etc.) | rejected | cancelled. Workforce requests submitted → approved → completed|pending_manual_application|failed.

**Cross-impacts:** loan issue GL Dr receivable / Cr funding (+schedule); repayment Dr Cash|Bank / Cr receivable; lifecycle completion mutates employee/salary records; corrections call markAttendance; attendance+leave feed payroll. Granular `hr.*` permission codes override via role_permissions.

**Routes:** /admin/employees · attendance · shifts · leave · designations · departments · loans · recruitment · performance · hr/compliance · hr/lifecycle(/$id) · hr/workforce-requests · /admin/my.

## 11. Payroll (`payroll`) — PAYROLL_ROLES + `hr.payroll.*` codes

**Entities & fields**
- Settings{salary_expense/employee_payable/deduction_liability accounts, working days 1–31, hours 1–24, OT multiplier 0–10}.
- Pay component{code 1–40, name 1–160, type earning|deduction, calculation fixed|percentage|per_day|per_hour|overtime|manual, defaults, taxable, affects_net_pay, GL accounts, active}.
- Salary structure{employee, effective_from, base 0–1e9 (req), frequency ≤20, currency ≤10, standards, components[≤50]{component, amount|percentage|rate, notes}}.
- Period{name, start/end, payment_date} status open|processing|calculated|approved|posted|paid|closed. Run{period, name, branch?, department?, notes ≤500}.
- Manual adjustment{runEmployee, type earning|deduction, name, amount 0.01–1e8, note}. Approve{allowWarnings} / reopen / post{postingDate}. Salary payment{runEmployee, amount, date, method (def bank), bank_account, reference ≤120, notes ≤300}. Reversal request{run, reason ≤2000} → approve|reject → retry-apply. Payslip PDF (HR or own). Cost report by dimension.

**Flows:** run draft → calculate (snapshot: attendance summary + structure + components + loan-recovery deductions + warnings) → calculated → approved (reopen possible) → posted → per-employee payments (unpaid → partial → paid). Reversal submitted → approved → completed via compensating voucher (or pending_manual/failed).

**Cross-impacts:** post → journal payroll_run: Dr earning expenses (dept/CC split) · Cr deduction liabilities · Cr Employee Payable (net) · loan recovery credits receivable + subledger. Payment → journal payroll_payment Dr Payable / Cr bank-or-cash account.

**Routes:** /admin/payroll · /admin/hr/payroll-ops (payslips, reversals, cost reports) · payslips in /admin/my.

## 12. CRM (`crm`) — manual §CRM, Segments & Analytics

**Entities & fields**
- Customer edit (MANAGERS): full_name ≤120, phone ≤40, email ≤160, city ≤80, notes ≤1000, tags[≤30×≤40], group. Extended profile: company, customer_code, DOB, gender, source, status, assigned employee. Consent: marketing/email/sms/whatsapp consents + suppression (audited variant exists).
- Tags{name 1–60, color ≤20; archive; assign/unassign}; Notes{1–2000}; Groups (master data).
- **Segments:** name 1–120, description ≤500, rule{match all|any, conditions[≤20]{field ≤40, op eq|neq|gt|gte|lt|lte|in|contains|is_true|is_false, value}}; preview; Audiences{name; members add[1–5000]/remove}.
- **Activities:** type (task|follow_up|meeting|visit|call_reference|email_reference|complaint|other), subject 1–240 (req), description ≤4000, priority low|normal|high, assignee, due_at; status open|in_progress|completed|cancelled.
- **Integration hub** (ADMINS): systems{voxodial|custom, name, enabled, refs, secrets}; mappings{external↔local, status suggested|confirmed|conflict|disabled}; inbound event envelope (voxodial.call.completed v1, voxodial.consent.updated v1).

**Flows & impacts:** Customer 360 merges orders, returns, notes, restaurant activity, loyalty, financials, duplicate candidates; consent drives campaign reachability + history; activities live in crm_activities; interactions timeline records loyalty/campaign events.

**Routes:** /admin/crm · crm-analytics · segments · growth · integrations · customers · customer-groups.

## 13. Loyalty (`loyalty`)

**Entities & fields:** program{name, enabled, points_per_currency, redeem_points_per_currency, min_redeem_points, max_redeem_percent, earn_on_tax, rounding, tier_basis}; tiers{name 1–60, threshold, multiplier 0–100, sort}; rewards{name 1–120, type fixed_discount|percentage_discount|free_item|bonus_points, value, points_cost, active}; manual adjust (MANAGERS){customer, points ≠0, reason 1–300}; redemption quote (read-only); claims{customer, reward, expiresInDays 1–3650} → used|cancelled|expired{reason}; expiry policy{enabled, expiry_days 1–3650, grace 0–365, effective_from} + apply (ADMINS); tier processing.

**Flows & impacts:** ledger types earn|redeem|adjust|reversal|expire. Earn on POS sales and restaurant settlements (idempotent); refunds reverse proportionally; expiry writes expiration lines + ledger; tier history recorded; interactions logged. **Claims do not deduct points** (see §16).

**Routes:** /admin/loyalty (+ per-customer on /admin/crm).

## 14. Marketing (`marketing`)

**Entities & fields:** promotion{name, type percentage|fixed|min_spend_percentage|min_spend_fixed, value, min_order_amount, max_discount_amount, channels, segment/audience, loyalty_only, active, window, usage limits}; coupon{code (uppercase unique), promotion, active, window, usage/per-customer limits, min_order, channels} + validate{code, orderAmount, channel, customerId}; campaign{name, objective, channel email|sms|whatsapp, segment/audience, promotion/coupon, subject, body, schedule_at} status draft|scheduled|sending|completed|cancelled; social post{channel, caption, media, scheduled_at, campaign} status …|demo_published; **free-gift campaign** (ecommerce license + marketing on){name 1–120, description ≤1000, active, priority, channels, window} + slabs{min_order_amount, gift_count ≥1, variant_ids[≥1], enabled, sort}.

**Flows & impacts:** campaign execute snapshots the consent-filtered audience into recipients as "sent" (**demo only — no provider wired**) + interaction rows + stats; social publishing is demo. Free gifts actually apply: zero-price lines on online orders with stock/COGS posting. Coupons/promotions: see §16 — not applied at checkout today.

**Routes:** /admin/growth · campaigns · promotions (Promotions & coupons / Free gifts) · social.

## 15. Shared core (all shops)

- **Settings sections** (`/admin/settings`; owner/admin for financial-ish sections, managers for ops): business, localization, tax, receipt, pos (incl. discount_roles, per-role payment_methods), inventory (negative stock, oversell roles), customers (credit), delivery (zones/slabs), pricing, purchasing, payments (bank details shown at checkout), expenses (approval threshold), documents, hours, notifications, labels. ⚠ several are stored but not yet read — see §16.
- **Modules:** business type retail|restaurant|trading|distribution|services|general + 14 flags (ADMINS); defaults per type.
- **Team:** invite{email, role, branch?} pending → accepted|revoked; member update{role, branch, isActive}; last active owner protected. **Audit:** filterable + CSV (ADMINS).
- **Files:** public product-images (≤10 MB, proxied /api/public/media) · private payment-proofs (≤5 MB), hr-documents, procurement-documents. Storage hard-capped (trial 500 MiB / licensed 10 GiB / per-shop override).
- **Data Manager** (ADMINS): 77 allow-listed tables (75 editable; shop_users + audit_logs read-only); condition builder (9 ops, ALL/ANY); preview-count-guarded bulk update/delete (≤50k, typed DELETE); JSON/CSV insert ≤1000; FK dependency resolver; owner-only typed RESET (transactional|catalog|everything). `/admin/uat-data` tenant clear/delete.
- **AI:** assistant (members) & salesman (public, 20/min/IP){message ≤2000, conversation, locale}; knowledge docs{title ≤200, content ≤200k}; business memory{scope, content ≤4000; approve}; approvals pending → executed|failed|rejected; permission modes disabled|approval|automatic per capability (create_purchase_order, adjust_inventory, record_expense, record_payment, create_supplier_invoice, generate_report, draft_message); alert types incl. low_stock, cash_discrepancy, receivables_outstanding, new_order.
- **BI:** executive dashboard{from/to, branch, compareMode (6 options)}; Reports Center — 26 registry reports across sales/inventory/accounting/finance/purchasing/crm/restaurant/payroll; export requires a paid license.
- **Subscription & usage** (/admin/subscription): plan + licenses; quota keys users, branches, registers, products, variants, customers, employees, orders_monthly, ai_requests_monthly, knowledge_documents, storage (**enforced: users, knowledge_documents, storage**); resource usage (storage hard; database, AI, network, members, MAU, customers; trial caps 500 MiB / 10k / 200 / 1 GiB / 5 / 100 / 1k).
- **Platform console** (/platform): plans (+features/limits), subscriptions & lifecycle (START_TRIAL … CHANGE_PLAN + schedules), entitlement/quota overrides, module licenses (grant modules|org|pack × 1–60 months|custom|lifetime; revoke; packs), tenant resource limits, enter-shop traversal (audited), platform admins.
- Misc: email branding + delivery log; 4 locales (en/ar/ur/hi); CSV exports; business events SaleCompleted/OrderCreated/PriceChanged.

---

## 16. Current boundaries & honest limitations (verified in code, 2026-10-03)

Do not promise these until built/fixed; several have fix candidates:

1. **Coupons/promotions are not applied at checkout** (POS, online or restaurant) — `validateCoupon` has no caller; pricing uses only `prices(type=sale)`. They exist as records + free-gift campaigns DO apply.
2. **Loyalty redemption isn't wired** — earn works (POS + restaurant settle); `redeemForOrder` has no caller and claims don't deduct points. Online orders/proof payments don't earn.
3. **No customer-group pricing** despite the group field; **staff cannot create customers** from POS/CRM (online checkout/sign-up/Data Manager only) and `credit_limit` has no edit UI.
4. **Table-QR ordering and loan issuing need seeded settings**: no UI writes `restaurant_settings.qr_*` or `payroll_settings.*account_id` — loans can't be issued until those accounts are set (UAT seed does it).
5. **KDS "sent" tickets** can't be advanced from the KDS screen (transition map lacks `sent`; Restaurant-POS `setKotStatus` can) — likely bug.
6. **Many settings persist but are not read** (pricing caps, PO approval, document prefixes, hours, notifications, several POS flags); the read ones are listed in §15.
7. **POs are client-created as `submitted`** (no approval path except via procurement awards); GRN posts no GL (value hits GL at invoice post — by design, document it).
8. **License-key mismatches** (vouchers→ecommerce, departments→accounting, restaurant setup/reservations/KDS key splits, promotions ecommerce-vs-marketing) — see the agent guide §6.2.
9. **Online orders have no cancel** (labels/payment/returns/hard-delete only).
10. **Manufacturing = planning only** (no production posting); **campaign sending & social publishing are demo-only** (no provider).

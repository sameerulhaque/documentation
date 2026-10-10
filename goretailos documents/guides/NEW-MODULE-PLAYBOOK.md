# GoRetailOS — New-Module Playbook (fully detailed)

> **Created 2026-10-02** as the complete, self-sufficient recipe for adding a module to GoRetailOS —
> follow it top-to-bottom and nothing is missed. Architecture rationale + verbatim code patterns:
> **`GORETAILOS_AGENT_GUIDE.md`** (same folder; its §1 rules and §3 templates are binding).
> Current modules (14): `retail_pos, restaurant_pos, restaurant_tables, restaurant_kitchen,
> ecommerce, inventory, purchasing, procurement, accounting, employees, payroll, crm, loyalty,
> marketing`.

> ## ⛔ LIVE-PRODUCTION SAFETY
> Live paying shops exist (esp. **Trendies / trendiestore.com**). Develop and verify against the
> UAT shop only; storefront/domain changes must tolerate the pre-migration schema (legacy-shape
> fallbacks); quota/licensing machinery fails open; `LICENSING_ENFORCED` stays OFF in production.

**Conventions:** `<mod>` = your snake_case module key (e.g. `rentals`), `<Mod>` = PascalCase.
Domain code = a `src/lib/<mod>.functions.ts` + `src/lib/<mod>.server.ts` pair (+ `<mod>-calc.ts`
for pure math). Never hand-edit `src/routeTree.gen.ts`.

---

## Step 0 — Design decisions BEFORE any code

| Decision | Rule |
|---|---|
| Entities + lifecycle statuses | `text` + **named** CHECK constraints; list the transitions |
| Money / derived values | computed in `<mod>-calc.ts` (pure, tested) or SQL — never the browser |
| Which roles touch what | pick from `ADMINS / MANAGERS / INVENTORY_ROLES / FINANCE_ROLES / SELLING_ROLES / PAYROLL_ROLES` per endpoint |
| Respect the `/admin/modules` toggle? | almost always yes → plan the toggle check (guide §1.3) alongside the license guard |
| Grant class per table | SERVER_ONLY (default) / AUTHENTICATED member-read / PUBLIC — per root `AGENTS.md` |
| Public storefront surface? | yes → Step 9 (site_key extension) + Step 10 (SEO/domains) |
| Countable commercial entity? | yes → Step 12 quota wiring |
| New `Permission`? | needs the union AND `ROLE_MATRIX` in `ShopContext.tsx` |
| Default-on for which business types? | you'll encode it in `moduleDefaultsFor` |

---

## Step 1 — Register the module key (ONE registry + its hand-maintained mirrors)

**1.1 The registry — `src/lib/module-keys.ts`:** add `'<mod>'` to `MODULE_KEYS`, a
`{ key, label, hint }` row to `MODULE_LABELS` (the `/admin/modules` checkbox list and the console
grant dialogs grow automatically from it), and a boolean to **`moduleDefaultsFor`'s `base` object**
(TypeScript forces this) + per-business-type defaults.

**1.2 The mirrors (hand-maintained — miss one and something half-works):**

| Mirror | What to do |
|---|---|
| `src/lib/saas-entitlement-calc.ts` `SAAS_MODULE_FEATURE_KEYS` | add `module.<mod>`-style key; **its test asserts `.length` (14 today) — update `saas-entitlement-calc.test.ts`** |
| `src/components/admin/LicenseGate.tsx` `ROUTE_MODULE_MAP` | add `["/admin/<mod>", "<mod>"]` — most-specific prefixes FIRST (segment-boundary matching) |
| `src/components/admin/nav.ts` | the Step-5 nav group/items carry `module: "<mod>"` tags |
| `src/lib/ai-registry.ts` | AI tools for the domain carry `module: "<mod>"` (Step 11) |
| `src/lib/uat-seed-spec.ts` `UAT_ENABLE_MODULES` | add it if UAT seeding should enable it |
| `src/lib/bi-report-contracts.ts` `BiReportModule` | only if the module ships BI reports |
| `supabase` grandfathering | the applied backfill migration enumerates the OLD keys only → **new modules are NOT grandfathered**. Decide the posture (Step 13); granting live shops lifetime for it = a new forward migration |

DB columns `module_licenses.module_key` / `module_packs.module_keys` have no CHECK — the app
validates via `MODULE_KEYS`, so no migration is needed for the key itself.

## Step 2 — Schema migration (one file; guide §3.1 template)

- Tables: `id uuid primary key default gen_random_uuid()` · `shop_id uuid not null references
  public.shops on delete cascade` (tenant wipe relies on the cascade) · named CHECKs ·
  `create index if not exists <t>_shop_idx(shop_id, …)` · timestamps · `create table if not exists`.
- **Same migration:** RLS enabled + policies + explicit Data-API grants per class, with the
  one-line intent comment (root `AGENTS.md`; templates in
  `docs/supabase-migration/DATA_API_GRANT_CONVENTION.md`). SERVER_ONLY is the default for domain
  tables (no anon/authenticated grants; `service_role` CRUD).
- `updated_at`: set explicitly in the server layer (designations style) or attach `set_updated_at`
  with the existence-guarded trigger block (`20261001090000` style).
- `src/integrations/supabase/types.ts` will be stale until regenerated — use the untyped handle
  `const sdb = supabaseAdmin as unknown as { from: (t: string) => any };` meanwhile.

## Step 3 — Server layer (the functions/server split; guide §3.2–3.3 verbatim)

- `src/lib/<mod>.functions.ts`: `shopScoped` zod + per-endpoint `.extend()`; an
  `assert…` helper built from `moduleGuards("<mod>")` (license) **plus the toggle check**
  (`assertModuleUsable(shopId, "<mod>")` or the `crm.functions.ts` `assertModule` pattern);
  handlers dynamically import the server module. GET reads / POST writes.
- `src/lib/<mod>.server.ts`: `supabaseAdmin` queries always `.eq("shop_id", shopId)`; referential
  pre-checks; `audit({ shopId, userId, action: "<entity>.<verb>", … })` on significant writes;
  friendly `throw new Error(...)` messages (unique-violation translation like `designations.server`).
- Pure math → `src/lib/<mod>-calc.ts` + `<mod>-calc.test.ts` (vitest picks up `src/**/*.test.ts`).
- Search params into `.or(...)` → `likeTerm` (`src/lib/utils.ts`).

## Step 4 — Panel (components/admin/<Mod>Panel.tsx; guide §3.4 idiom)

`useServerFn` + `useQuery({ queryKey: ["<mod>", shopId], enabled: Boolean(shopId) })`; writes are
plain async handlers with `busy`/`error`/`notice` state then `invalidateQueries`; **inline
messages, never toasts** (`bg-destructive/10` error strip; emerald success text); `Panel`,
`TableWrap`/`Th`/`Td`, `Pill`, `EmptyState`, `FormModal`, `btn.*`, `Field`/`inputClass`,
`Spinner`/`TableSkeleton`, `Pagination`, `SearchSelect` from the AdminUI kit; `window.confirm` for
destructive confirms; **no `window.prompt()`** — use a small form/modal for input.

## Step 5 — Route, navigation, gating

- **Route file:** `src/routes/_authenticated/admin.<mod>.tsx` (flat dot-names; `admin.<mod>.<sub>.tsx`
  for children, `.index.tsx`, `.$id.tsx`):
  ```tsx
  export const Route = createFileRoute("/_authenticated/admin/<mod>")({
    head: () => ({ meta: [{ title: "<Mod> — Retail OS" },
      { name: "description", content: "…" }] }),
    component: <Mod>Page,
  });
  function <Mod>Page() {
    const { shop, role } = useAdminShop();
    return (<div className="grid gap-6"><PageHeader title="<Mod>" description="…" />
      {shop ? <<Mod>Panel shopId={shop.id} role={role} /> : null}</div>);
  }
  ```
- **Nav:** a `NavGroup` in `components/admin/nav.ts` with `module: "<mod>"` on the group and items
  (+ `permission:` tags). ⚠️ icons are string names — add the lucide import AND the `ICONS` map
  entry in `AdminSidebar.tsx`, or it silently falls back to LayoutDashboard. `NAV_FLAT` then feeds
  breadcrumbs + ⌘K automatically (⌘K ignores module toggles — known gap).
- **Gating:** the `ROUTE_MODULE_MAP` entry (Step 1.2) gives the license lock screen; the toggle is
  enforced server-side (Step 3), not at the route.

## Step 6 — Enablement (nothing to build — verify the chain)

`/admin/modules` renders one toggle per `MODULE_LABELS` row automatically; `updateShopModules`
(ADMINS) stores explicit overrides into `shops.modules` jsonb and audits `shop.modules_updated`;
defaults come from `moduleDefaultsFor(shops.business_type)`; the sidebar hides disabled modules
(`moduleEnabled`), licensed-but-disabled shows nothing, enabled-but-unlicensed shows a Lock badge.

## Step 7 — Offline/POS interaction (only if the module touches selling)

The offline sales queue (`sync.functions.ts`) is **licensing-exempt** by design (business
continuity). If your module affects POS flows, keep its reads cheap and never add a hard gate into
the sync path.

## Step 8 — Storefront data surface (if the module feeds the EXISTING storefront)

If the module only adds data the current storefront shows (like products/categories), follow the
existing storefront functions' conventions: slug-resolved, published-only, allow-listed columns,
null-safe. Fallback imagery: `storefront/image-fallbacks.ts` pictograms or verified real artwork
only — most of `public/theme/assets/images` is grey "WxH" placeholder files.

## Step 9 — A NEW per-shop customer SITE (restaurant-website pattern)

Extending the sites system (`shop_site_*`) to a new `site_key` touches **a forward migration + six
hard-coded code points**:

1. **Migration:** widen the `site_key` CHECK on ALL FOUR tables (`shop_site_settings`,
   `shop_site_banners`, `shop_site_navigation`, `shop_site_pages`) — the CHECKs are inline/unnamed
   (`<table>_site_key_check` by default; verify in `pg_constraint`): drop + re-add each as
   `check (site_key in ('restaurant','<new>'))`. `shop_domains.site_key` has no CHECK — no change.
2. `src/lib/shop-site.functions.ts` — `siteKeySchema = z.enum(["restaurant","<new>"])` + the
   guard's `moduleGuards("<mod>")`.
3. `src/lib/shop-site.server.ts` — `SHOP_SITE_KEYS` + the site's data bundle in `getPublicShopSite`.
4. `src/router.tsx` — the `homePath` mapping for the new site's root path.
5. `src/routes/sitemap[.]xml.ts` — a `<new>SiteEntries(shopId)` helper + the host branch
   (`tenant.siteKey === "<new>"`).
6. `src/routes/_authenticated/admin.storefront.tsx` **DomainsTab "Serves" select** — add the
   `<option>` and widen `onSetSite`'s type. ⛔ never change the selector on live shops' domains.
7. Admin editor route (copy `admin.restaurant-site.tsx`, `SITE_KEY = "<new>"`) + public routes
   (copy `store.$shopSlug_.dine.tsx` + its `p.$pageSlug` — the trailing `_` un-nests from the
   storefront layout) + a shell component (copy `components/restaurant-site/shell.tsx`;
   strict-hex `--rsite-*` vars, colour ladder site → storefront theme → default).

No `start.ts` change needed (the 301 exemption covers any non-null `site_key`). Any new TOP-LEVEL
public path (outside `/store/...`) must join `RESERVED_PREFIXES` in `src/lib/domain-routing.ts` —
exact entries (e.g. `/robots.txt`, not `/robots`), or custom-domain prefix rewriting breaks it.

## Step 10 — SEO (every public page)

SSR the data in the loader; `head` builds title/description/OG + **one canonical** via
`canonicalUrl(primaryDomain, slug, subpath)` with the ladder site-domain → storefront domain →
`SITE_ORIGIN`; typed JSON-LD (`Restaurant`, `Product`+`Offer`, …); missing/unpublished ⇒
`robots: noindex`. Dynamic `robots[.]txt.ts`/`sitemap[.]xml.ts` only — **never add static
`public/robots.txt`/`sitemap.xml`** (they shadow the routes). Layout routes never stamp a canonical
on child pages (one canonical per page — the storefront layout gates on the deepest match).

## Step 11 — AI wiring (three places)

1. **Capability catalog** — `src/lib/ai-registry.ts`: `AI_TOOLS` entries
   (`name "<domain>.<verb>", title, purpose, domain, module: "<mod>", roles, mutates, risk`);
   extend the `AiToolDomain` union for a new domain. `ai-policy.ts` `canUseTool` then enforces
   role + module toggle (+ `INTENT_KEYWORDS` for free-text routing).
2. **Read handlers** — `ai-orchestrator.server.ts` `READ_HANDLERS["<domain>.<verb>"]` (a registered
   tool without a handler politely answers "not available yet").
3. **Writes** — `ai-actions.server.ts`: an `ACTION_ROLES` entry + zod schema + `executeAiAction`
   switch case, and the matching enum value + description in `ai.server.ts`'s
   `request_action_approval` tool. All writes go through the approval flow; tool calls log to
   `ai_tool_calls`; chat turns meter via `ai_messages` (the usage screen's AI count).

## Step 12 — Quotas & commercial limits

Countable entity the plans console should cap → FOUR touches: a `QUOTA_REGISTRY` entry
(`saas-quota-calc.ts`), a `deriveUsage` case (`saas-usage.server.ts`, `countHead(table, q =>
q.eq("shop_id", shopId)…)`), the **hand-maintained key list** in `PlatformConsole.tsx` `Plans()`,
and `await assertWithinQuota(shopId, "<key>")` at the write boundary (+ `ENFORCED_QUOTA_KEYS` if it
blocks; precedence: `tenant_limit_overrides` > plan > unlimited; fails open; shops with no
subscription are never limited). No migration needed — `quota_key` has no CHECK.
Also: heavy tables → `tenant_db_records` forward migration; new storage buckets → the capped-bucket
set in `tenant_storage_used_bytes` + `storage_quota_allows` AND `TENANT_BUCKETS`
(`tenant-admin-spec.ts`); app-served transfers → `recordUsage(shopId, "network", bytes)`;
`usage-limits.ts` is **byte-identical with VOXOPlan** — tier caps change in both repos + SQL
together. Optional: `data-manager.server.ts` `TABLE_SPECS` + `tenant-admin-spec.ts`
`REPRESENTATIVE_TABLES` entries for the new tables.

## Step 13 — Licensing posture (decide, don't drift)

New modules are **not grandfathered** — existing shops show the module as unlicensed (invisible
while `LICENSING_ENFORCED` is OFF, lock badge when ON). Decide with the platform owner: pack it,
promotional grants, or a one-off grandfather forward migration for existing shops. The console
panels, grant dialog, packs and `/admin/subscription` licenses card all pick the key up
automatically from `MODULE_KEYS`/`MODULE_LABELS`.

## Step 14 — Platform console & BI (optional)

BI reports → `bi-report-contracts.ts` `BiReportModule` + the bi-reports role resolution; new
platform-console surfaces follow `PlatformConsole.tsx` + `components/platform/*Panel.tsx` idioms
(AdminUI kit, never-vanish error states).

## Step 15 — Tests, gates, documentation

1. **Gates (no CI — run them yourself):** `npx tsc --noEmit` = 0 · `npx vitest run` green ·
   `npx vite build` green · routeTree regenerated.
2. **Tests:** `<mod>-calc.test.ts` for the pure math; Playwright spec in `tests/e2e/` if there's a
   public surface (filename prefix picks the project: `visitor-`, `storefront-`, `owner-`, `pos-`,
   `admin-`).
3. **Docs:** section + Δ-table row in `docs/testing/SUMMARY.md` (⛔ rules
   apply) and a phase entry in `GORETAILOS_UAT_COMPLETION_PLAN.md`; a `USER_MANUAL.md` section
   (manuals are `?raw`-imported — edit in place, never move); `docs/operations/SUMMARY.md` if the
   commercial posture is unusual; `SHARED_OS_UI_DESIGN_SYSTEM.md` (BOTH repo mirrors,
   byte-identical) only for genuinely new UI patterns.

---

## Master definition of done (tick all)

**Registration (Step 1):**
- [ ] `MODULE_KEYS` + `MODULE_LABELS` + `moduleDefaultsFor.base` (+ business-type defaults)
- [ ] Mirrors: `SAAS_MODULE_FEATURE_KEYS` (+ its length test) · `ROUTE_MODULE_MAP` (specific-first) ·
      nav `module:` tags + `ICONS` entry · `ai-registry` tags · `uat-seed-spec` ·
      (`bi-report-contracts` if BI)

**Vertical (Steps 2–6):**
- [ ] Migration: shop-FK cascade tables + named CHECKs + indexes + RLS + per-class grants with
      intent comments (same file)
- [ ] functions/server split: `moduleGuards("<mod>")` + toggle check + `.eq("shop_id", …)` +
      audits + bounded zod + `likeTerm`
- [ ] `<mod>-calc.ts` + test for pure math
- [ ] Panel (inline messages, data states, AdminUI kit) + route (`— Retail OS` title) + nav group
- [ ] `/admin/modules` toggle verified end-to-end (enable → nav appears → disable → server fns refuse)

**Platform wiring (Steps 7–14):**
- [ ] POS/offline untouched or exempt-safe · [ ] storefront/site extension complete (all 7 points,
      Step 9) + `RESERVED_PREFIXES` · [ ] SEO: SSR + one canonical + JSON-LD + sitemap branch ·
- [ ] AI: registry + read handlers + action map (if applicable) ·
- [ ] Quotas: registry/deriveUsage/console list/assert (if countable); db-records + buckets +
      `recordUsage` joined · [ ] licensing posture decided (not grandfathered by default)

**Proof (Step 15):**
- [ ] tsc 0 · vitest green · vite build green · ⛔ no live shop touched
- [ ] UAT playbook + completion plan + USER_MANUAL updated

---

## Appendix — complete file inventory for a new module `<mod>`

| Kind | Path |
|---|---|
| Migration(s) | `supabase/migrations/<ts>_<mod>.sql` (+ forward: grandfather decision, `tenant_db_records`, buckets, `site_key` CHECK if a new site) |
| Server | `src/lib/<mod>.functions.ts` · `src/lib/<mod>.server.ts` · `src/lib/<mod>-calc.ts` (+ `.test.ts`) |
| UI | `src/components/admin/<Mod>Panel.tsx` · `src/routes/_authenticated/admin.<mod>*.tsx` |
| Registry edits | `module-keys.ts` · `saas-entitlement-calc.ts` (+ test) · `LicenseGate.tsx` · `nav.ts` + `AdminSidebar.tsx` ICONS · `ai-registry.ts` · `uat-seed-spec.ts` · (`bi-report-contracts.ts`) |
| New site (optional) | `shop-site.functions/server.ts` edits · `router.tsx` homePath · `sitemap[.]xml.ts` branch · `admin.storefront.tsx` Serves option · `admin.<new>-site.tsx` · `store.$shopSlug_.<new>.tsx` (+ `p.$pageSlug`) · `components/<new>-site/shell.tsx` · `domain-routing.ts` RESERVED_PREFIXES |
| AI (optional) | `ai-registry.ts` · `ai-orchestrator.server.ts` READ_HANDLERS · `ai-actions.server.ts` · `ai.server.ts` enum |
| Quotas (optional) | `saas-quota-calc.ts` · `saas-usage.server.ts` · `PlatformConsole.tsx` Plans list · `tenant-admin-spec.ts` · `data-manager.server.ts` |
| Docs/tests | UAT playbook + completion plan + `USER_MANUAL.md` · `tests/e2e/<prefix>-<mod>.spec.ts` |

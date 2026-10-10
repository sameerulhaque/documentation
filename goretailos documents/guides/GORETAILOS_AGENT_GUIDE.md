# GoRetailOS — Agent Build Guide

**Audience:** an AI agent (or engineer) working on GoRetailOS. This is the canonical how-to for
(1) working on the product safely — **there are LIVE, PAYING customers** — (2) the architecture
patterns every change must follow, and (3) building a **new** module beyond the current fourteen.
The complete ordered recipe is **`NEW-MODULE-PLAYBOOK.md`** in this folder.

**Companion docs (read alongside):**
- Root `AGENTS.md` — Lovable git rules + the **binding migration grant discipline** (full templates:
  `docs/supabase-migration/DATA_API_GRANT_CONVENTION.md`).
- `../operations/SUMMARY.md` (licensing section) — licensing, resource quotas, the Supabase
  billing-meter mapping. `../operations/SUMMARY.md` (restaurant website) — the per-site public pattern.
- `../../SHARED_OS_UI_DESIGN_SYSTEM.md` — the UI spec (byte-identical mirror with VOXOPlan;
  GoRetailOS exceptions are recorded inside it: inline messages instead of toasts, admin has no RTL
  `dir` flip).
- `../testing/SUMMARY.md` (testing rules & checklist)
  · `../manuals/USER_MANUAL.md`
  (**imported with `?raw` by routes — never move/rename the manuals folder**).
- `src/routes/README.md` — routing conventions.

---

## 0. Orientation (read first)

- **What it is:** a multi-shop retail/restaurant ERP. The tenancy unit is **`shops.id`** — every
  tenant row carries `shop_id`. One user ↔ many shops via `shop_users`
  (roles `owner|admin|manager|cashier|accountant|inventory|staff`).
- **Modules (14):** `retail_pos, restaurant_pos, restaurant_tables, restaurant_kitchen, ecommerce,
  inventory, purchasing, procurement, accounting, employees, payroll, crm, loyalty, marketing` —
  single source `src/lib/module-keys.ts` (`MODULE_KEYS`, `MODULE_LABELS`, `BUSINESS_TYPES`,
  `moduleDefaultsFor`). ⚠️ several **hard-coded mirrors** exist — see §6.1.
- **Stack:** TanStack Start (SSR) + Supabase (Postgres + Auth + Storage), React Query, Tailwind v4.
  No edge functions — server logic is `createServerFn` handlers + `*.server.ts` modules.
- **Enablement:** two columns on `shops` — `business_type text` + `modules jsonb` (explicit
  overrides only; defaults come from `moduleDefaultsFor(business_type)`); resolved by
  `getShopModules` (`modules.server.ts`), toggled on `/admin/modules` (`ModulesPanel`, ADMINS only).
- **Licensing:** `module_licenses` (shop_id, module_key; `period_end NULL` = LIFETIME; live shops
  grandfathered). Ships dark behind `LICENSING_ENFORCED` (**OFF in production, never flip it
  there**) — guards log `[licensing would-block]` instead.

### ⛔ LIVE-PRODUCTION SAFETY (overrides everything)
Live customer shops exist (esp. **Trendies / `trendiestore.com`**). Never enter, modify, license,
or test against a live shop; never change `trendiestore.com`'s Domains "Serves" selector; UAT work
uses the dedicated UAT shop / `ZZ-` disposables only; quota/limit machinery must **fail open**;
storefront changes must be verified against the old schema too (deploy-ordering fallbacks).

---

## 1. Non-negotiable rules

1. **Every tenant table** has `shop_id uuid not null references public.shops on delete cascade`
   (tenant deletion relies on the cascade), RLS enabled **and its Data-API grants declared in the
   same migration** per root `AGENTS.md` (classes: PUBLIC / AUTHENTICATED / SERVER_ONLY —
   SERVER_ONLY is the default for new domain tables / NOT_DATA_API_EXPOSED; one intent comment per
   grant block; forbidden: `grant all`, blanket grants; corrections are forward-only migrations).
2. **Guards before any query.** Every authenticated fn: `requireMembership` / `requireRole`
   (`guards.server.ts`; role sets `ADMINS`, `MANAGERS`, `INVENTORY_ROLES`, `FINANCE_ROLES`,
   `SELLING_ROLES`, `PAYROLL_ROLES`) — or the licensed variants from
   `moduleGuards("<key>")` (`licensed-guards.server.ts`). Platform admins resolve as a synthetic
   owner inside `requireMembership` (audited traversal).
3. **`moduleGuards` checks the LICENSE only — not the on/off toggle.** A module fn that must
   respect `/admin/modules` ALSO checks the toggle: `assertModuleUsable(shopId, key)`
   (`licensing.server.ts` — toggle then license) or the `crm.functions.ts` `assertModule` pattern.
4. **Every query filters `.eq("shop_id", shopId)`** on top of RLS; the server file works through
   `supabaseAdmin` ONLY after the functions file verified membership/role.
5. **Money & derived values** are computed by DB/server, never the browser; pure business math
   lives in a `<domain>-calc.ts` with a co-located `<domain>-calc.test.ts` (vitest only picks up
   `src/**/*.test.ts`).
6. **Public (anon) surfaces** take a `slug`/hostname, never a `shopId`; return published,
   allow-listed columns only; never throw raw internals; tolerate pre-migration schemas (legacy
   column fallbacks in the domain resolvers).
7. **Search terms that feed PostgREST `.or(...)` go through `likeTerm`** (`src/lib/utils.ts`) —
   raw `, ( )` fail the parse (proven live, PGRST100 400).
8. **Errors**: `throw new Error("plain sentence")`; surface with `getErrorMessage` (zod-aware).
   **UI feedback is INLINE messages, never toasts** (sonner exists but is unmounted — documented
   exception); `window.confirm` for destructive confirms is the house style, but **no
   `window.prompt()`** for input (harness/embedded browsers block it) — use a small form/modal.
9. **tsconfig traps** (`strict` + extras): `noPropertyAccessFromIndexSignature` → bracket access
   (`values["key"]`, `process.env["X"]`); `noUncheckedIndexedAccess` → `list[0]!` or guards;
   `exactOptionalPropertyTypes` → optional inputs typed `?: T | null | undefined` and conditional
   spreads `...(x ? { k: x } : {})`.
10. **Generated artifacts:** never hand-edit `src/routeTree.gen.ts` (dev server regenerates);
    `src/integrations/supabase/types.ts` is stale for new tables — newer modules use
    `const sdb = supabaseAdmin as unknown as { from: (t: string) => any };`.
11. **Gates before hand-off:** `npx tsc --noEmit` = 0 · `npx vitest run` green (879+) ·
    `npx vite build` — there is NO CI for the frontend; run them yourself.

---

## 2. Architecture map (where things live)

| Concern | File(s) |
|---|---|
| Auth middleware | `src/integrations/supabase/auth-middleware.ts` (`requireSupabaseAuth`) |
| Guards + role sets | `src/lib/guards.server.ts`; licensed variants `src/lib/licensed-guards.server.ts` (`moduleGuards("<key>" \| "ANY")`) |
| Module registry | `src/lib/module-keys.ts` (+ the §6.1 mirror list) |
| Enablement | `shops.business_type` + `shops.modules` jsonb; `modules.server.ts` / `modules.functions.ts`; UI `components/admin/ModulesPanel.tsx`; client `ShopContext.tsx` (`moduleEnabled`/`moduleLicensed`, `Permission` union + `ROLE_MATRIX`) |
| Licensing | `licensing.server.ts` (state, `assertModuleLicensed/Usable/AnyLicense/ExportAllowed`), `saas-licenses.server.ts` (grants/packs), console `components/platform/PlatformConsole.tsx`, route gate `components/admin/LicenseGate.tsx` (`ROUTE_MODULE_MAP`) |
| Commercial quotas | `saas-quota-calc.ts` (`QUOTA_REGISTRY`, `ENFORCED_QUOTA_KEYS`) + `saas-usage.server.ts` (`deriveUsage`, `assertWithinQuota`); plans/limits console in `PlatformConsole.tsx` |
| Resource usage | `usage-limits.ts` (**byte-identical mirror with VOXOPlan — change both + SQL together**), `usage.functions.ts` (`getShopResourceUsageFn`, `assertUploadWithinQuotaFn`, `recordUsage`), `platform-limits.functions.ts`, panels `ResourceLimitsPanel.tsx` / `SubscriptionPanel.tsx` |
| Navigation | `components/admin/nav.ts` (`NAV_GROUPS` with `module:`/`permission:` tags; `NAV_FLAT` feeds breadcrumbs + ⌘K); icons = string names mapped in `AdminSidebar.tsx` `ICONS` |
| Admin UI kit | `components/admin/AdminUI.tsx` (PageHeader, Panel, Tabs, Field/input classes, `btn.*`, Pill, EmptyState, Table*, FormModal, ExportButton, Spinner/Skeletons, Pagination, MobileCards) + `SearchSelect.tsx` |
| Storefront & sites | `store.$shopSlug.*` routes (+ `store.$shopSlug_.<site>` un-nested sites), `shop-site.functions/server.ts`, `components/restaurant-site/`, `storefront/image-fallbacks.ts` |
| Domains | `src/lib/domain-routing.ts` (`storePrefixRewrite`, `RESERVED_PREFIXES`), `domains.server.ts`, `src/start.ts` (301 exemption for `site_key` domains), `__root.tsx` `__TSR_DOMAIN_SITE__` |
| SEO | `src/lib/seo.ts` (`SITE_ORIGIN`, `canonicalUrl`, `primaryDomainFromMatches`, `absUrl`, `metaText`); dynamic `robots[.]txt.ts` + `sitemap[.]xml.ts` (never static files in `public/`) |
| AI | `ai.server.ts` (SALESMAN/ASSISTANT_TOOLS + switches), `ai-actions.server.ts` (`ACTION_ROLES` + `executeAiAction`), `ai-registry.ts` (`AI_TOOLS` capability catalog), `ai-orchestrator.server.ts` (`READ_HANDLERS`), `ai-policy.ts` (`canUseTool`, `INTENT_KEYWORDS`), `ai-knowledge.server.ts` |
| Offline POS | `sync.functions.ts` — **licensing-exempt** (business continuity); POS terminals queue offline |
| Data tooling | `data-manager.server.ts` `TABLE_SPECS` allowlist; `tenant-admin-spec.ts` (`TENANT_BUCKETS`, `REPRESENTATIVE_TABLES`) |

---

## 3. Canonical patterns (copy these; `designations` is the model domain)

### 3.1 Migration (from `20260926120000_manufacturing_bom.sql`)
Tables: `id uuid pk default gen_random_uuid()` · `shop_id` FK cascade · named CHECKs for statuses ·
`create index if not exists <t>_shop_idx on public.<t>(shop_id, …)` · `created_at/updated_at` ·
`create table if not exists`. Then the grant/RLS block per class (AUTHENTICATED member-read shown;
SERVER_ONLY = no policy, service_role grant only):

```sql
do $$ declare t text; begin
  foreach t in array array['<mod>_<a>','<mod>_<b>'] loop
    execute format('alter table public.%I enable row level security', t);
    -- Data API: anon=none · authenticated=read · service_role=CRUD (AUTHENTICATED member-read)
    execute format('drop policy if exists "shop members read %1$s" on public.%1$I', t);
    execute format('create policy "shop members read %1$s" on public.%1$I for select to authenticated using (public.is_shop_member(shop_id))', t);
    execute format('grant select on public.%I to authenticated', t);
    execute format('grant select, insert, update, delete on public.%I to service_role', t);
  end loop;
end $$;
```
`updated_at`: either set it explicitly in the server layer (`designations.server.ts` style) or
attach `set_updated_at` with the existence-guarded trigger block (`20261001090000` style).

### 3.2 Functions file (`<mod>.functions.ts`) — thin, guarded, validated
```ts
const shopScoped = z.object({ shopId: z.string().uuid() });
async function assertManage(context: { supabase: unknown; userId: string }, shopId: string) {
  const { requireRole, MANAGERS } = (await import("./licensed-guards.server")).moduleGuards("<key>");
  await requireRole(context.supabase as never, shopId, context.userId, MANAGERS);
  // + toggle check if the module must respect /admin/modules (rule §1.3)
}
export const create<Entity> = createServerFn({ method: "POST" })
  .middleware([requireSupabaseAuth])
  .inputValidator((raw: unknown) => shopScoped.extend({ <entity>: <entity>Input }).parse(raw))
  .handler(async ({ data, context }) => {
    await assertManage(context, data.shopId);
    const { create<Entity>: run } = await import("./<mod>.server");
    return run(data.shopId, context.userId, data.<entity>);
  });
```
GET for reads, POST for writes; strings `.max()`-bounded; ids `z.string().uuid()`; dynamic imports
for guards + server module.

### 3.3 Server file (`<mod>.server.ts`) — supabaseAdmin + audit
```ts
import { supabaseAdmin } from "@/integrations/supabase/client.server";
import { audit } from "./audit.server";
export async function create<Entity>(shopId: string, userId: string, input: <Entity>Input) {
  // referential pre-checks, then:
  const { data, error } = await supabaseAdmin.from("<mod>_<entity>")
    .insert({ shop_id: shopId, ...mapped, created_by: userId } as never).select("id").single();
  if (error) throw new Error(error.message);
  await audit({ shopId, userId, action: "<entity>.created", entity: "<entity>", entityId: data.id, data: {…} });
  return { id: data.id };
}
```
Audit action names: `<entity>.<verb>`. Input types `?: T | null | undefined` (tsconfig).

### 3.4 Panel + page (inline-message idiom)
Panel: `useServerFn` + `useQuery({ queryKey: ["<mod>", shopId], enabled: Boolean(shopId) })`;
writes = plain async handlers with `busy`/`error` state → `invalidateQueries`; render
`{error ? <p className="mb-3 rounded-md bg-destructive/10 px-3 py-2 text-sm text-destructive">{error}</p> : null}`;
loading `<Spinner/>` centered; `EmptyState` for zero rows; `Panel`/`TableWrap`/`Pill`/`btn.*` from
AdminUI. Page: `src/routes/_authenticated/admin.<mod>.tsx` with
`head: () => ({ meta: [{ title: "<Page> — Retail OS" }, { name: "description", … }] })`, body =
`PageHeader` + `{shop ? <Panel shopId={shop.id} role={role}/> : null}` via `useAdminShop()`.

### 3.5 Public site/storefront pattern
Public fns take `slug` (no auth middleware), return null-safe published data only
(`getPublicShopSite` never throws). Site routes un-nest from the storefront layout with the
`store.$shopSlug_.<site>.tsx` trailing-underscore convention; `head` builds title/OG/canonical
(`canonicalUrl(primaryDomain, slug, subpath)` ladder: site domain → storefront domain →
`SITE_ORIGIN`) + JSON-LD; missing site ⇒ `robots: noindex`.

---

## 4. Cross-cutting subsystems (wire these for EVERY module)

Compact contract — templates and the complete registration list live in
`NEW-MODULE-PLAYBOOK.md`:

- **Registration:** `module-keys.ts` (+ `moduleDefaultsFor`'s `base` object — the compiler forces
  it) AND the hard-coded mirrors in §6.1 — `SAAS_MODULE_FEATURE_KEYS` (+ its length test),
  `ROUTE_MODULE_MAP`, nav `module:` tags + `ICONS`, `ai-registry` tags, `uat-seed-spec`.
- **Licensing:** guard files with `moduleGuards("<key>")`; add the toggle check where
  `/admin/modules` must be respected; new modules are **NOT grandfathered** (the applied backfill
  migration enumerates the old 14) — decide the commercial posture; grandfathering live shops for a
  new module = a new forward migration.
- **Quotas:** countable entity → `QUOTA_REGISTRY` + `deriveUsage` case + the hand-maintained
  `Plans()` key list in `PlatformConsole.tsx` + `assertWithinQuota` at the write boundary (+
  `ENFORCED_QUOTA_KEYS` if it blocks). Heavy tables → `tenant_db_records` forward migration; new
  buckets → the capped set + `TENANT_BUCKETS`; app-served bytes → `recordUsage`.
- **Public surface:** extending the per-shop **sites** system means a forward migration widening
  the `site_key` CHECK on all four `shop_site_*` tables + the half-dozen hard-coded `'restaurant'`
  sites in code (playbook Step 9 lists every one) + a sitemap branch + `RESERVED_PREFIXES` for any
  new top-level public path.
- **SEO:** SSR loaders, ONE canonical via the §3.5 ladder, JSON-LD, dynamic robots/sitemap only.
- **UX invariants:** inline messages (no toasts) · `getErrorMessage` · `likeTerm` · no `prompt()` ·
  usage/limits panels never vanish on failure · storefront fallback imagery = real artwork or the
  neutral pictograms (`storefront/image-fallbacks.ts`), never the theme pack's grey "WxH" files.
- **Docs/tests:** UAT playbook section + Δ row + completion-plan phase; `USER_MANUAL.md` section;
  calc+test pair; gates green.

---

## 5. Definition of done (per module)

- [ ] All §6.1 registration points + mirrors updated (the `SAAS_MODULE_FEATURE_KEYS` length test
      will fail until you do).
- [ ] Migration per §3.1 with the correct grant class; RLS on; cascade delete verified.
- [ ] Functions/server split per §3.2–3.3: licensed guards + toggle check + `shop_id` filters +
      audit entries; inputs zod-bounded.
- [ ] Panel/page per §3.4 (inline errors, data states, `— Retail OS` titles); nav group + icons;
      `ROUTE_MODULE_MAP` prefix (most-specific first).
- [ ] Public/SEO/domains wired if customer-facing (§4); `RESERVED_PREFIXES` updated for new
      top-level paths.
- [ ] Quotas/licensing posture decided and wired; AI tools registered if applicable.
- [ ] `<domain>-calc.test.ts` for pure math; Playwright spec if it has a public surface
      (`visitor-`/`storefront-` filename prefixes).
- [ ] Gates: tsc 0 · vitest green · vite build green · routeTree regenerated.
- [ ] Docs: UAT playbook + completion plan + USER_MANUAL (+ design-system mirrors only for NEW UI
      patterns, kept byte-identical).

---

## 6. Known sharp edges (keep these aligned; don't re-introduce)

### 6.1 The module-key mirrors (single-source is aspirational — these are hand-maintained)
`src/lib/module-keys.ts` is the registry, but these copies must be edited with it:
`saas-entitlement-calc.ts` `SAAS_MODULE_FEATURE_KEYS` (+ the `.length` assertion in its test) ·
`components/admin/LicenseGate.tsx` `ROUTE_MODULE_MAP` · `components/admin/nav.ts` `module:` tags ·
`ai-registry.ts` `AI_TOOLS` module tags · `uat-seed-spec.ts` `UAT_ENABLE_MODULES` ·
`bi-report-contracts.ts` `BiReportModule` (BI subset) · `modules.functions.ts` business-type enum.
DB `module_licenses.module_key`/`module_packs.module_keys` have NO CHECK — the app validates.

### 6.2 Current inconsistencies (fix opportunistically, never copy)
- `vouchers.functions.ts` guards with `ecommerce` but lives under `/admin/accounting`
  (map says `accounting`); `/admin/promotions` nav=`marketing` vs map=`ecommerce`;
  `/admin/growth` nav=`marketing` vs map=`crm`; restaurant reservations/waitlist/kds nav vs map
  disagree (`restaurant_pos` vs `restaurant_tables`/`restaurant_kitchen`).
- `/admin/restaurant-site` is missing from `ROUTE_MODULE_MAP` (segment matching means `/admin/
  restaurant` does not cover it).
- The ⌘K menu filters by permission only — disabled modules' pages still appear there.
- Several HR function files check only the license, not the toggle.
- `ai_requests_monthly` counts the `ai_usage` table, which nothing writes — the real AI meter is
  `ai_messages` (30d) on the usage screen.

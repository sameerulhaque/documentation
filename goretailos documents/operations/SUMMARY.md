# Operations (single source)

Replaces the former operations documents (originals in git history).

## Module licensing & super admin
- Table `module_licenses` (shop_id, module_key; `period_end NULL` = lifetime). Live shops are
  grandfathered.
- Switch `LICENSING_ENFORCED` (production rule: see the agent guide); when off, guards only log
  `[licensing would-block]`. Code: `src/lib/licensing.server.ts`, `licensed-guards.server.ts`.
- Grant licences on `/platform`; a shop sees its status on `/admin/subscription`. Resource quotas
  (storage etc.) are shown in the platform resource-limits panel.

## Email and Google sign-in after moving to Supabase Pro
Auth emails and Google sign-in broke after the move. Fix order: set up the sending domain DNS
(Resend) first → configure the Send Email Hook → re-register Google sign-in (can run in parallel)
→ restore the delivery-log panel → test sign-up, reset, magic link, invite → clean up.

## Restaurant website
Each shop can have a second, restaurant-only customer site with its own settings and theme, a live
menu, and its own domain assigned to that site.

## Connecting your own domain (Namecheap)
1. Add the domain in the store dashboard. 2. Add the DNS records shown there at Namecheap.
3. Wait for verification and go live (usually minutes, up to 48 h). If it fails, recheck records
and remove conflicting ones.

## Data loading (history)
Phase 8 moved lists to server-side search and paging; completed.

## Rules
Live-shop safety rules: `docs/guides/GORETAILOS_AGENT_GUIDE.md` §1.

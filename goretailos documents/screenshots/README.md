# Documentation Screenshots

Real screenshots captured from the live GoRetailOS application (`https://goretailos.com`)
using Playwright. These images are referenced by the user-facing documentation
(`USER_MANUAL.md`, `usage.md`, `POS_MANUAL.md`, `LABELS_MANUAL.md`).

## Capture details

- **Source**: Live production site `https://goretailos.com`
- **Shop**: Trendies Store (Haroon Shopping Center)
- **Currency**: PKR · **Timezone**: Asia/Karachi
- **Viewport**: 1440 × 900
- **Tooling**: `tests/e2e/docs-screenshots.spec.ts` + `playwright.screenshots.config.ts`
- **Re-capture**: `npx playwright test --config=playwright.screenshots.config.ts`

## Manifest

| Screenshot | Route | Used In | Description |
| --- | --- | --- | --- |
| `auth/staff-sign-in.png` | `/admin/sign-in` | USER_MANUAL, usage | Staff sign-in screen for the admin portal |
| `admin/dashboard.png` | `/admin` | USER_MANUAL, usage | Owner dashboard with KPIs and recent activity |
| `catalog/products-list.png` | `/admin/products` | USER_MANUAL, usage | Product catalog list with stock & pricing |
| `inventory/stock-movements.png` | `/admin/inventory` | USER_MANUAL | Stock movement ledger (in/out/adjust) |
| `purchasing/purchase-orders.png` | `/admin/purchasing` | USER_MANUAL | Purchase orders list and statuses |
| `accounting/reports-ledger.png` | `/admin/accounting` | USER_MANUAL | Accounting ledger / reports |
| `customers/customers-list.png` | `/admin/customers` | USER_MANUAL | Customer directory with balances |
| `settings/shop-settings.png` | `/admin/settings` | USER_MANUAL | Shop profile & configuration |
| `admin/team-members.png` | `/admin/team` | USER_MANUAL | Team members & role management |
| `ai/business-assistant.png` | `/admin/ai` | USER_MANUAL | AI business assistant chat |
| `admin/data-manager.png` | `/admin/data` | USER_MANUAL | Data import/export manager |
| `pos/terminal.png` | `/admin/pos` | USER_MANUAL, POS_MANUAL | POS terminal with cart & keypad |
| `storefront/home.png` | `/store/{slug}` | USER_MANUAL, usage | Storefront home page |
| `storefront/shop.png` | `/store/{slug}/shop` | USER_MANUAL, usage | Storefront product grid |
| `storefront/product-detail.png` | `/store/{slug}/product/{slug}` | USER_MANUAL, usage | Product detail page |
| `storefront/cart.png` | `/store/{slug}/cart` | USER_MANUAL, usage | Shopping cart |
| `storefront/checkout.png` | `/store/{slug}/checkout` | USER_MANUAL, usage | Checkout form |
| `storefront/track-order.png` | `/store/{slug}/track-order` | USER_MANUAL | Order tracking |
| `labels/overview.png` | `/admin/labels` | LABELS_MANUAL | Barcodes & Labels page header |
| `labels/label-design.png` | `/admin/labels` | LABELS_MANUAL | Label design panel — templates, size, fields |
| `labels/sheet-setup.png` | `/admin/labels` | LABELS_MANUAL | Pre-cut sheet setup (ruler-friendly geometry) |
| `labels/select-variants.png` | `/admin/labels` | LABELS_MANUAL | Selecting variants & copies |
| `labels/preview.png` | `/admin/labels` | LABELS_MANUAL | Label preview |
| `labels/sheet-preview.png` | `/admin/labels` | LABELS_MANUAL | Full pre-cut sheet preview |
| `labels/json.png` | `/admin/labels` | LABELS_MANUAL | Template JSON editor |

## Notes

- Screenshots contain real demo data from the owner's shop (Trendies Store).
- Re-capture any image after significant UI changes to keep docs accurate.
- Keep filenames lowercase with hyphens; update this manifest when adding images.
# GoRetailOS documentation — start here

One page per folder. Each folder has a `SUMMARY.md` covering every document inside it. When two older
documents disagreed, the code was checked and the summary follows the code.

## Root-level documents
| File | What it is |
| --- | --- |
| `README.md` | Project overview, how to run it, first-run guide and troubleshooting. |
| `AGENTS.md` | Binding rules for anyone changing code (git, database grants, ERP/SSO decisions). |
| `SHARED_OS_UI_DESIGN_SYSTEM.md` | **The** UI standard (shared with VOXOPlan). |
| `roadmap.md` | Live task list — what is done and what waits on the owner. |


## Folders
| Folder | Summary | In one line |
| --- | --- | --- |
| `guides/` | [SUMMARY](guides/SUMMARY.md) | How to work on the code safely; how to add a module. Read first. |
| `product/` | [SUMMARY](product/SUMMARY.md) | What the product does, who it's for, what's built. |
| `manuals/` | [SUMMARY](manuals/SUMMARY.md) | User, POS and labels manuals (shown inside the app — don't move). |
| `integration/` | [SUMMARY](integration/SUMMARY.md) | Single sign-on and the ERP connection to GoCoreOS (release 1.16.0). |
| `operations/` | [SUMMARY](operations/SUMMARY.md) | Licensing, email, domains, restaurant website. |
| `supabase-migration/` | [SUMMARY](supabase-migration/SUMMARY.md) | Database grant rules and the move to Supabase Pro. |
| `testing/` | [SUMMARY](testing/SUMMARY.md) | Test playbooks, results and safety rules. |
| `expansions/` | [SUMMARY](expansions/SUMMARY.md) | History of HR, CRM, procurement, restaurant, BI, SaaS builds. |
| `design-system/` | [SUMMARY](design-system/SUMMARY.md) | Older design specs (current one is at the root). |
| `alignment/` | [SUMMARY](alignment/SUMMARY.md) | Records of aligning UI with the shared standard. |
| `dotnet-migration/` | [SUMMARY](dotnet-migration/SUMMARY.md) | Paused .NET rewrite plan — not used by the live app. |
| `archive/` | [SUMMARY](archive/SUMMARY.md) | Old handoffs, history only. |
| `screenshots/` | `screenshots/README.md` | Images used by the manuals. |

## Key facts
Modules, roles and safety rules: `guides/GORETAILOS_AGENT_GUIDE.md`. ERP/SSO status and release: `integration/SUMMARY.md` §9.

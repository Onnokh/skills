---
name: rankstas-paradise
description: Query and update the SEO plans and Search Console results for Sleevy and Missingmounts via the Ranksta’s Paradise MCP. Use when the user asks how either site's pages or keywords are performing, which pages to optimize or create, whether pages are indexed, what the SEO plan/registry contains, what was published or changed and when — and after changing either site's title, content, or internal links (log the Action).
---

# Ranksta’s Paradise (MCP)

Ranksta’s Paradise is the source of truth for the SEO plans (keyword registry with intent and rationale per page) and Search Console outcomes for both configured sites. Answer SEO questions from its output; never query Google or scrape Search Console yourself.

Access it through its **MCP tools** — `status`, `pages`, `page`, `queries`, `opportunities`, `registry`, `log`, `history`, and the write tools `registry_add`, `registry_set`, `log_add`. Each returns one JSON document.

Every tool takes a **`site`** argument: a configured site ID, not a URL — `sleevy` (`https://sleevy.app`) or `missingmounts` (`https://missingmounts.com`). There is **no default; always pass `site`.** Data is served from the hosted deployment, kept current by a daily server-side sync — you never fetch from Google yourself.

## Question → tool

| Question | Tool |
| --- | --- |
| What pages performed well? What needs optimizing? | `pages` (optional `window`) — every page with window deltas, plan context, signals, and a `verdict` + `reasons` |
| Why is this page flagged? Deep dive | `page` (`path`) — plan, rationale, daily series, top queries, baseline, and logged actions |
| Where are the opportunities? What should we make or improve? | `opportunities` (optional `kind`: `striking-distance`\|`ctr`\|`new-demand`\|`cannibalization`) |
| What is the plan? What was set out, and when? | `registry` (targets, intents, phases, publish dates) and `log` (interventions) |
| What queries are we seen for? | `queries` (optional `page`, `minImpressions`, `includeBrand`, `limit`, `window`) |
| Are our pages indexed? | `registry` or `pages` — each target carries `indexed` (`indexed`/`not-indexed`/`unknown`) |
| How much traffic over time? | `history` (optional `limit`) — daily true site totals |
| Is the data fresh? | `status` |

## Reading the output

- `verdict` and `reasons` are already judged by the tool's classifier — relay them; recompute nothing.
- `trueTotals` is the real traffic (query-less daily totals). `nonBrand`/`allQueries` come from query rows, which Google strips of anonymized long-tail queries, so they undercount volume — use them for keyword progress and query analysis, `trueTotals` for "how much traffic".
- `indexed` comes from Google's URL Inspection API, refreshed on each sync (`inspectedAt` says when). A `not-indexed` target explains zero visibility before any content diagnosis — check it first when a page shows `no-visibility`.
- Every response embeds its window bounds; quote dates when reporting numbers.

## Logging work

Every entry in the log is either an **Action** or a **Note**. An Action is a concrete change to a page; the tool centres a before/after window on its date, so logging it is what lets a later readout show whether the change moved the numbers. A Note is a free-form observation — *not* a change — and gets no readout.

Call `log_add` with `{ site, path, kind, date?, note? }`.

- **Kind.** Actions: `publish`, `content-update`, `title-change`, `internal-links`, `consolidation`. Observation: `note`. Reach for `note` only when nothing on the page changed; otherwise pick the Action kind that matches the change.
- **Path, not keyword.** The log attaches to a target page by path, and many keyword rows can share one page — so log the page you changed, never a single keyword.
- **Date.** Defaults to today; pass `date` with the day the change actually shipped so the readout's window lines up with it.
- Log an Action on your own initiative right after changing a page's title, content, or internal links — that is the whole reason later readouts can compare before and after.

## Writing back

- New keyword mapping (e.g. acting on a `new-demand` opportunity): `registry_add` with `{ site, target, keyword, cluster, intent, priority, why }` (omit `keyword` for an inventory-only page row).
- Edit existing rows with `registry_set` — `{ site, target, keyword?, patch: { … } }`. Patch fields: `cluster`, `intent`, `country`, `priority`, `publishedAt`, `baselineDate`, `status`, `whyOpportunity`, `newTargetUrl`. Without `keyword` every row of the target is patched; with it, only that row. `newTargetUrl` remaps rows to another page (consolidating cannibalized keywords). Fields must not contain commas; the tool validates and refuses duplicates.
- **Confirm with the user before editing the registry** — it is the plan of record.
</content>

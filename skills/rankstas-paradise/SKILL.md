---
name: rankstas-paradise
description: Query and update the SEO plans, keyword demand, and Search Console results for Sleevy, Missingmounts, Printfeest, and Shadertown via the Ranksta's Paradise MCP. Use when the user asks how a site's pages or keywords are performing, which pages to optimize or create, whether pages are indexed, how much search volume a keyword has, what the SEO plan/registry contains, what was published or changed and when — and after changing a site's title, content, or internal links (log the Action).
---

# Ranksta's Paradise (MCP)

Ranksta's Paradise is the source of truth for the Registry (the SEO plan: keywords mapped to target URLs with intent and rationale), the Search Console outcomes, and the keyword demand behind the plan. Answer SEO questions from its output; never query Google or scrape Search Console yourself.

Access it through its **MCP tools**. Each returns one JSON document.

Every tool takes a **`site`** argument: a configured site ID, not a URL. There is **no default; always pass `site`.**

| Site ID | Property |
| --- | --- |
| `sleevy` | `https://sleevy.app` |
| `missingmounts` | `https://www.missingmounts.com` |
| `printfeest` | `https://www.printfeest.nl` |
| `shadertown` | `https://www.shadertown.com` |

Data is served from the hosted deployment, kept current by a daily server-side sync — you never fetch from Google yourself.

## Question → tool

| Question | Tool |
| --- | --- |
| What pages performed well? What needs optimizing? | `pages` (optional `window`) — every page with window deltas, plan context, signals, and a `verdict` + `reasons` |
| Why is this page flagged? Deep dive | `page` (`path`) — plan, rationale, daily series, top queries, baseline, and logged actions |
| Where are the opportunities? What should we make or improve? | `opportunities` (optional `kind`: `striking-distance`\|`ctr`\|`new-demand`\|`cannibalization`; optional `limit`, default 50) — opportunities come strongest-first, and `totalSignals` says how many matched beyond the limit |
| What is the plan? What was set out, and when? | `registry` (targets, intents, phases, publish dates) and `log` (Actions and Notes) |
| Does the plan aim at searches that exist? | `registry_health` — every planned Keyword with a demand `verdict`. The one report a site with no visibility yet can still act on |
| What else is worth targeting? | `keywords_proposed` (free), then `keywords_discover` to find more (**costs money** — see below) |
| What queries are we seen for? | `queries` (optional `page`, `minImpressions`, `includeBrand`, `limit`, `window`) |
| Are our pages indexed? | `registry` or `pages` — each target carries `indexed` (`indexed`/`not-indexed`/`unknown`) |
| How much traffic over time? | `history` (optional `limit`) — daily true site totals, with visits per day |
| What happened today? | `today` — the only report that carries the day in progress, sales included |
| Who is on the site right now? | `live`, `live_events` |
| What did visitors do? What did they buy? | `events`, `revenue` |
| Is the data fresh? | `status` |

## The Market: check it before you believe a volume

Every keyword number a site reports is measured in one **Market** — a DataForSEO country and language. A Market is a property of the **Site**, not of a Keyword row, and it **defaults to the United States in English**. A site that does not sell into the US in English must say so, or its whole plan is measured against the wrong country and reads as no demand.

- `registry_health`, `registry`, `queries`, `opportunities` and `keywords_proposed` all name the `market` their numbers describe. **Read it first.** A Dutch site reporting `United States / en` is misconfigured, not undemanded — a real case moved from 10 to 2,920 monthly searches on that one setting.
- `market.provider` says `labs` or `google-ads` **in advance**. Google-Ads countries (49 of them) report no `difficulty` and no `intent` at all, so an absent difficulty there is the country, not an error.
- A wrong Market is a **Site setting** to correct, not a Registry row to patch. Say so and confirm with the user, then re-read `registry_health`: the metrics store is keyed by market, so a corrected Market starts with an empty cache and every planned Keyword is re-asked at the vendor.

Two absences mean different things, and neither is a zero: **no `demand` key** means nothing was asked (no key, no sync, or a brand/operator query that is never worth paying for); a **null `searchVolume`** means DataForSEO was asked and reports the term as too rare, which is not the same as nobody searching it.

## Reading the output

- `verdict` and `reasons` are already judged by the tool's classifier — relay them; recompute nothing.
- `trueTotals` is the real traffic (query-less daily totals). `nonBrand`/`allQueries` come from query rows, which Google strips of anonymized long-tail queries, so they undercount volume — use them for keyword progress and query analysis, `trueTotals` for "how much traffic".
- `totals.monthlyVolume` is the size of the addressable market — every search, **not** a traffic forecast, and not the share a first-page ranking would win.
- `difficultyGap` is a keyword's difficulty minus the site's Domain Rating. Two vendors' scales, so read it as a rough guide and never as a verdict.
- **Never sum word-order variants.** DataForSEO reports `traktatie 1 jaar` and `1 jaar traktatie` at the same volume because they are one clustered demand. Adding them up multiplies one audience. Dedupe to distinct concepts before you total anything.
- `indexed` comes from Google's URL Inspection API, refreshed on each sync (`inspectedAt` says when). A `not-indexed` target explains zero visibility before any content diagnosis — check it first when a page shows `no-visibility`.
- Every response embeds its window bounds; quote dates when reporting numbers.

## Finding new keywords (this one costs money)

`keywords_discover` expands one seed at DataForSEO. It bills about **$0.0001 per row returned plus $0.01 per task**, so `limit` is the price of the call — leave it at the default 200 unless a narrower run came back almost empty. It **stores nothing**.

**You are the relevance judge.** Every filter the tool applies is numeric or structural (`minVolume`, `maxDifficulty`, `intents`, the brand test, the already-known test), so none of them knows what the site is about. Read every row and keep only what is about this site's subject:

- A word with a second meaning is a word two audiences search, and the biggest numbers are often the worst rows. Check the seed's words for other meanings before you believe a volume.
- Watch for a **second audience on the same noun**: seeding `knijpfruit` (a printable wrapper) returned mostly supermarket shopping demand for the fruit pouch itself — real volume, wrong customer.
- Drop what the site cannot legally or practically sell: licensed characters, physical goods for a printables site, and so on.
- `source: "related"` is the noisiest for exactly the reason it is useful — its answers need not contain the seed.

Then `keywords_propose` stores the rows you kept as **Proposals** for a person to review. It is **free** and asks the vendor nothing, so one run is paid for however many rows you keep. Pass the rows back **unchanged** from the discover result — the numbers on a Proposal are the vendor's, frozen as they read at the run, which is why a Proposal may disagree with `registry_health`'s current metric. Both are true.

- Plan discovery and proposal as **one pass**: a Proposal needs the full row, so re-reading a printed summary means paying for the seed twice.
- Read the drop counts before re-running a seed. An empty result with a high `droppedKnown` means the seed is exhausted, not that the subject has no demand.
- A Proposal is **not** a Registry row. `registry_add` accepts one; `keywords_dismiss` sets it aside, and a dismissed keyword stays dismissed so later runs on the same seed stay useful.

## Logging work

Every entry in the log is either an **Action** or a **Note**. An Action is a concrete change to a page; the tool centres a before/after window on its date, so logging it is what lets a later readout show whether the change moved the numbers. A Note is a free-form observation — *not* a change — and gets no readout.

Call `log_add` with `{ site, path, kind, date?, note? }`.

- **Kind.** Actions: `publish`, `content-update`, `title-change`, `internal-links`, `consolidation`. Observation: `note`. Reach for `note` only when nothing on the page changed; otherwise pick the Action kind that matches the change.
- **Path, not keyword.** The log attaches to a target page by path, and many keyword rows can share one page — so log the page you changed, never a single keyword.
- **Date.** Defaults to today; pass `date` with the day the change actually shipped so the readout's window lines up with it.
- Log an Action on your own initiative right after changing a page's title, content, or internal links — that is the whole reason later readouts can compare before and after.

## Writing back

- New keyword mapping (e.g. acting on a `new-demand` opportunity or an agreed Proposal): `registry_add` with `{ site, target, keyword, cluster, intent, priority, why }` (omit `keyword` for an inventory-only page row).
- Edit existing rows with `registry_set` — `{ site, target, keyword?, patch: { … } }`. Patch fields: `cluster`, `intent`, `priority`, `publishedAt`, `baselineDate`, `status`, `whyOpportunity`, `newTargetUrl`. Without `keyword` every row of the target is patched; with it, only that row. `newTargetUrl` remaps rows to another page (consolidating cannibalized keywords). Fields must not contain commas; the tool validates and refuses duplicates.
- There is **no `country` on a Registry row.** The market a row's numbers describe belongs to the Site — two rows of one site cannot be in different markets. Set the Site's Market instead.
- **Confirm with the user before editing the registry** — it is the plan of record.

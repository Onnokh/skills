---
name: rankstas-paradise
description: Query and update the SEO plans and Search Console results for Sleevy and Missingmounts via the local Ranksta’s Paradise CLI. Use when the user asks how either site's pages or keywords are performing, which pages to optimize or create, whether pages are indexed, what the SEO plan/registry contains, what was published or changed and when — and after changing either site's title, content, or internal links (log the intervention).
---

# Ranksta’s Paradise CLI

The tool at `/Users/onnokleinhofmeijer/Documents/rankstas-paradise` is the source of truth for the SEO plans (keyword registry with intent and rationale per page) and Search Console outcomes for both configured sites. Answer SEO questions from its output; never query Google or scrape Search Console yourself.

Run every command as:

```
cd /Users/onnokleinhofmeijer/Documents/rankstas-paradise && bun run seo <command> --site <site-id> [options]
```

`--site` is a configured site ID, not a URL: use `sleevy` for `https://sleevy.app` or `missingmounts` for `https://missingmounts.com`. It defaults to `sleevy`. The API exposes the same catalog at `GET /api/sites`. Each command prints exactly one JSON document on stdout (bun echoes the command line to stderr — parse stdout only). Errors exit non-zero with the message on stderr. `bun run seo help` lists all options.

## Question → command

| Question | Command |
| --- | --- |
| What pages performed well? What needs optimizing? | `pages` — every page with window deltas, plan context, signals, and a `verdict` + `reasons` |
| Why is this page flagged? Deep dive | `page </path>` — plan, rationale, daily series, top queries, baseline, and logged actions |
| Where are the opportunities? What should we make or improve? | `opportunities [--kind striking-distance\|ctr\|new-demand\|cannibalization]` |
| What is the plan? What was set out, and when? | `registry` (targets, intents, phases, publish dates) and `log list` (interventions) |
| What queries are we seen for? | `queries [--page </path>] [--min-impressions N] [--include-brand]` |
| Are our pages indexed? | `registry` or `pages` — each target carries `indexed` (`indexed`/`not-indexed`/`unknown`) |
| Is the data fresh? | `status` |

## Reading the output

- `verdict` and `reasons` are already judged by the tool's classifier — relay them; recompute nothing.
- `trueTotals` is the real traffic (query-less daily totals). `nonBrand`/`allQueries` come from query rows, which Google strips of anonymized long-tail queries, so they undercount volume — use them for keyword progress and query analysis, `trueTotals` for "how much traffic".
- `indexed` comes from Google's URL Inspection API, refreshed on each sync (`inspectedAt` says when). A `not-indexed` target explains zero visibility before any content diagnosis — check it first when a page shows `no-visibility`.
- Every response embeds its window bounds and `mode` (`live`/`debug`); quote dates when reporting numbers.

## Writing back

- After changing a page on either site for SEO (title, content, internal links, consolidation), record it so future readouts can compare before/after: `log add --site <site-id> --path </path> --kind <publish|content-update|title-change|internal-links|consolidation|note> [--date YYYY-MM-DD] --note "<what changed>"`.
- New keyword mapping (e.g. acting on a `new-demand` signal): `registry add --site <site-id> --target </path> --keyword "<k>" --cluster "<c>" --intent <i> --priority P<n> --why "<rationale>"`.
- Edit existing rows with `registry set --site <site-id> --target </path> [--keyword "<k>"]` plus any field flag: `--cluster --intent --country --priority --published-at --baseline-date --status --why --new-target`. Without `--keyword` every row of the target is patched; with it, only that row. `--new-target </path>` remaps rows to another page (consolidating cannibalized keywords). Fields must not contain commas; the CLI validates and refuses duplicates. Confirm with the user before editing the registry — it is the plan of record.

## Data fetching

- Read commands touch only local data. `sync` calls Google (a daily launchd job also runs it); run it only when `status` shows stale dates. `backfill` is a one-time 16-month history fetch.
- `--debug` targets an isolated fake database — testing only, never for real answers.

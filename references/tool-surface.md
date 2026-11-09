# Tool surface

Reference file for the `screaming-frog-mcp` skill (CC BY 4.0 — Eoghan Henn and contributors). Every tool the server exposes, grouped by the job it does. Names are written
without the server prefix — match by suffix.

Load triggers: choosing a tool, or checking whether a capability exists before concluding it does not; starting a crawl with `sf_crawl` (authentication, config provenance, bounding it); moving bulk output outside the allowed base directory; deleting or cleaning up stored crawls

**Verification key.** ✅ = exercised directly and behaves as described here.
⬜ = present in the tool list but not yet exercised; the description is the
server's own, and its real behaviour, response shape and failure modes are
unknown. Do not present ⬜ behaviour as fact — try it, then report back so
this file can absorb the result.

## Contents

- Crawl lifecycle
  - Authentication for MCP-started crawls
  - config_path provenance
  - Bounding an MCP-started crawl
  - Deleting stored crawls
- Discovery — what can I ask for?
- Bulk data out
- Single-URL lookups
- Files and execution on the user's machine

---

## Crawl lifecycle

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_list_crawls` | `limit` defaults to 10. Returns `instanceDirName` (this is the crawl id), `url`, `spiderMode` (`Spider`\|`List`), `crawlName`, `urlsCrawled`, `percentageComplete`, `time`, `projectVersion`. `percentageComplete` is genuine progress — a low value means the crawl really is partial, **not** that it is unusable (see "Session-start protocol" in SKILL.md). |
| ✅ | `sf_load_crawl` | Takes `crawl_id` = the `instanceDirName` value. Returns a bare success string. Required before any query tool. Loaded state is **lost when the host app restarts**. **Refuses with `IllegalStateException: UNEXPECTED_MESSAGE loadCrawl in state: SpiderPausedState` when the app holds a paused crawl** — call `sf_crawl_progress` first; if it is already the crawl you want, you don't need to load at all. **Also fails with `Crawl is open in another SEO Spider instance`** when the crawl is open in the desktop UI — the user must close it there before the MCP can load it. |
| ✅ | `sf_crawl` | Starts a crawl. Takes `crawl_url` (a single start URL), optional `config_path`, `crawl_name`, `project_name`. **Spider semantics only — there is no list-mode parameter** (see `gotchas.md`; a List-mode crawl runs through the application's own CLI spawned from `sf_run_node_js_script` — `workflows.md`, "Re-checking a list of URLs"), and a config profile saved while the app was in List mode silently produces a one-URL spider crawl (see [config_path provenance](#config_path-provenance)). `config_path` accepts an absolute path outside the MCP's allowed base directory; **omitted, the crawl runs under whatever configuration the MCP's instance currently holds**, and nothing in the receipt says whether a URL limit applies — **do not assume one.** For any auth-protected target, read [Authentication for MCP-started crawls](#authentication-for-mcp-started-crawls) first. Starting a crawl evicts the currently loaded one and, on a large site, runs until you pause it — see [Bounding an MCP-started crawl](#bounding-an-mcp-started-crawl). While it runs, every query tool is refused with `SEO Spider is busy` (see `gotchas.md`). Do not fire this speculatively on a large site. |
| ✅ | `sf_crawl_progress` | Read-only and safe on a paused crawl, and still answers while a crawl is active — it is the counter to poll for a deliberate pause. Returns `databaseId` (which crawl is in memory), `completed`, `waiting`, `active`, `percentComplete`, `stateName`. **Call this before `sf_load_crawl`** — it tells you both what is loaded and how much of the site is actually covered — and **before any absence-based conclusion**: `waiting` is the queue, and the listing's rounded `percentageComplete` can read `100.0` over a non-empty one (see "Absence and subtraction traps" in `workflows.md`). **Instance-scoped:** it describes the crawl held by the MCP's own instance, not one running in the desktop UI — while a UI crawl runs it keeps reporting the MCP's previous crawl (e.g. `completed: 283, percentComplete: 100`), a confident status about the wrong job. For crawls started outside the MCP, poll `sf_list_crawls` and judge by `urlsCrawled` (see `gotchas.md`). |
| ✅ | `sf_pause_crawl` / `sf_resume_crawl` | Pause and resume. **Pausing does not release the instance.** `sf_load_crawl` on a paused crawl fails with `IllegalStateException: UNEXPECTED_MESSAGE loadCrawl in state: SpiderPausedState`, so pause-load-resume is not a handover route. The attempt is at least non-destructive: resuming afterwards restored the crawl with progress intact. **Pausing is, however, the only way to read anything from an MCP-started crawl in flight:** query tools are refused while the crawl is active, so the working loop is pause → query → resume, which behaved cleanly on every repetition. Resume is also the state-changing call to fear on a shared instance — it starts fetching the queue under any other session reading the paused crawl (see "Sharing one instance between sessions" in SKILL.md). |
| ✅ | `sf_clear_crawl` | Empties the instance's memory. **On a database-stored crawl it does not delete the project** — the crawl remained in `sf_list_crawls` with all 33,387 URLs and reloaded immediately. That makes it the release path for handing a paused crawl back to the desktop UI (`sf_load_crawl` being refused in `SpiderPausedState`). Confirm the crawl is a stored project before clearing, and re-check `sf_list_crawls` after. On a crawl with no durable home it is destructive. No tool deletes a stored crawl — see [Deleting stored crawls](#deleting-stored-crawls). |
| ⬜ | `sf_export_crawl` | Exports the crawl itself (as distinct from a report). |

### Authentication for MCP-started crawls

Three routes were tested against an HTTP-basic-auth-protected host; only one
survives a crawl started with `sf_crawl`.

- **Credentials in the start URL (`https://user:pass@host/`) — fails.**
  They authenticate the start URL itself (200, correct title), then the
  crawl dies there: the Spider rewrites every discovered internal link to
  carry the userinfo **lowercased**, mangling any mixed-case password, and
  no URLs are queued.
- **Credentials saved in the app (Configuration > Authentication) — fails.**
  They do not reach a crawl started through the MCP, with or without a
  `config_path`. Signature: start URL `Status Code: 0`, `Indexability
  Status: No Response`, raw `java.lang.NullPointerException` in the Status
  field — the app attempting to raise its interactive auth dialog with no UI
  to raise it in (see `gotchas.md`). Credentials being saved in the app is
  not evidence that an unattended crawl will authenticate.
- **HTTP header in the crawl config — works. Default to this for any
  auth-protected target.** Configuration > HTTP Header, add
  `Authorization: Basic <base64(user:pass)>`, save it in an ordinary crawl
  config profile, pass that profile as `config_path`. It lives in the
  ordinary config rather than the separate `.seospiderauthconfig`, so it
  travels with the profile into unattended runs.

The separate `.seospiderauthconfig` cannot be supplied at all: the MCP
exposes one config parameter where the CLI has two (see `gotchas.md`).

### config_path provenance

A config profile inherits the mode the desktop app was in when it was saved,
and nothing in the saved artefact reveals it. A profile saved in List mode
carries Store-without-Crawl settings on the Spider > Crawl tab (separate
**Crawl** and **Store** checkboxes per link type), and since `sf_crawl` is
spider-only, such a profile silently yields a crawl that records the start
URL's outlinks but queues none of them — a one-URL crawl with no error.
When a crawl completes at one URL, read `summary.totalUrlsEncountered` from
`sf_generate_report(category: "Crawl Overview")` before investigating
anything else — `gotchas.md` has the full diagnostic. Save (or re-save)
profiles intended for `sf_crawl` with the app in Spider mode.

### Bounding an MCP-started crawl

A crawl started through `sf_crawl` carries no bound you did not put there.
With no `config_path` it runs under the instance's current configuration,
and whether that configuration limits the URL count is not visible from the
receipt or from `sf_crawl_progress` — so on a large site the crawl is
bounded only by a deliberate `sf_pause_crawl`, and a pause that is merely
intended is not a bound. The failure shape: a crawl meant to pause at a
fixed count for comparability with earlier crawls is next looked at with a
multiple of that count fetched and a far larger queue, because throughput
rose steeply between the first reading and the next (on a large site,
roughly 400 URLs/min early on, roughly 1,000/min
later).

Three rules follow. **Key the bound on the crawl's own counter, not on the
clock:** poll `completed` from `sf_crawl_progress` and pause when it
crosses the target. **Treat the first throughput reading as a lower
bound** — the crawler warms up, so a polling interval sized on the first
reading overshoots. And **poll from the session that started the crawl**,
because while it runs nothing else on the instance can be queried (see
`gotchas.md`, "SEO Spider is busy").

### Deleting stored crawls

No MCP tool deletes a stored crawl: `sf_clear_crawl` empties the instance
and the project stays in `sf_list_crawls`. A script run through
`sf_run_node_js_script` can reach the store
(`~/.ScreamingFrogSEOSpider/ProjectInstanceData/<instanceDirName>/`), but
never delete or modify those folders from a script: the application keeps
a `DO_NOT_MODIFY_THESE_FILES.txt` there saying crawl data is managed only
from the SEO Spider, and a project index sits beside the crawl folders,
so a hand deletion bypasses the application's own bookkeeping (observed
on one install at last verification). Both cleanup routes are the
user's: delete crawls in the application's Crawls dialog, or set crawl
retention to delete automatically (a user-side setting, observed set to
never delete). The agent's share:

- Do not launch throwaway CLI runs: omitting `--save-crawl` does not keep
  a run out of the store — every CLI list run observed was stored regardless
  (`workflows.md`, Route A).
- At session end, name every stored crawl the session created, every CLI
  run included, by `crawlName`, `instanceDirName` and whether it is still
  needed, so the user can delete the rest in one pass.

Stored crawls and their by-products are what fills the disk under a long
CLI crawl; the free-disk guard and its recovery are in `workflows.md`,
Route A.

## Discovery — what can I ask for?

Data is organised as **SEO element × filter**, mirroring the desktop UI. The
names are not guessable and discovery is cheap, so resolve rather than assume.

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_list_available_filters_for_seo_element` | `seo_element_name` is a fixed enum (Internal, External, Response Codes, Canonicals, Directives, Hreflang, Links, Structured Data, Sitemaps, Page Titles, Validation, AI, Accessibility, …). Returns the filters valid for it. |
| ✅ | `sf_list_available_data_fields_for_seo_element_and_filter` | Returns the field names valid for that element *and* filter. **Field sets differ per filter** — this is the single most common source of avoidable errors. |
| ⬜ | `sf_list_available_reports` | Lists reports for `sf_generate_report`. |
| ⬜ | `sf_list_available_bulk_exports` | Lists exports for `sf_generate_bulk_export`. |

Useful filter sets observed:

- **Response Codes** — `Internal Blocked by Robots.txt`, `Internal Success (2xx)`, `Internal Redirection (3xx)`, `Internal Client Error (4xx)`, plus External and non-prefixed variants of each.
- **Canonicals** — `Canonicalised`, `Self Referencing`, `Non-Indexable Canonical`, `Missing`, `Multiple Conflicting`, `Unlinked`, `Outside <head>`.
- **Directives** — `Noindex`, `Index`, `Nofollow`, `NoArchive`, `Unavailable_After`, …
- **Internal** — `HTML`, `JavaScript`, `CSS`, `Images`, `PDF`, …

## Bulk data out

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_export_seo_element_urls` | The workhorse. NDJSON. **Supply `file_path` unless the result is genuinely tiny** — see "Getting data out without flooding the context" in SKILL.md, and the loop in `workflows.md`. `data_fields` omitted = all fields. Also takes `max_rows`, `start_index`, `segment_name`. Two traps: `max_rows` **caps the receipt's `total_records`**, so it cannot be used to probe a count; and an **empty filter throws** `IllegalArgumentException: start_index is greater than the total number of URLs of 0` rather than returning an empty set. |
| ✅ | `sf_generate_report` | `category` is colon-delimited (`Category:Subcategory`); supports `CSV`/`NDJSON` and `file_path`. **`category: "Crawl Overview"` is the population cross-check** — it carries totals the filters don't store (`Total External blocked by robots.txt`, `URLs Displayed`, content-type breakdowns), which is how you tell a genuine zero from an unpopulated filter. It also carries `summary.totalUrlsEncountered` — the first thing to read when a crawl completes at one URL (see `gotchas.md`). **Reports are not NDJSON:** the file is a single pretty-printed JSON document spanning many lines, whatever extension you asked for, so parse it with one `JSON.parse` of the whole file — the per-line idiom used for element exports throws a SyntaxError on line 1. Its `summary` block reconciles category totals in one read (internal non-indexable = blocked + noindex + redirect targets), which is often the whole cross-check. Some categories return "Report generated no data". |
| ✅ | `sf_generate_bulk_export` | Bulk export equivalent. Two exercised quirks (live 62.5k crawl): an **empty report returns the plain string `"Report generated no data"`** — no exception (unlike `sf_export_seo_element_urls`), no file written; check the result string before assuming a file exists, and apply the same true-zero-vs-unpopulated-filter caveat. And the receipt's **`path` field points to an internal temp file** (`…/temp/tmpCrawlOverviewReport`); the real saved location is only in the trailing "Export saved to …" line — read that, not `path` (see `gotchas.md`). **`category: "Queued URLs"` times out at the MCP layer on a large queue** (~1M rows), writes no partial file, and drops the loaded crawl with it — route large queue exports through the desktop UI (see `gotchas.md`). On a small queue (hundreds of rows) it returns fast and is safe even when heavier exports on the same crawl time out — which is what makes it usable as the absence guard in `workflows.md`. The same timeout-drops-the-crawl behaviour applies to any heavy bulk export (`Links: All Inlinks` on a ~100k-URL crawl reproduced it), and the drop hits every session sharing the instance. |
| ✅ | `sf_bulk_export_page_content` | Page content in bulk — assume very large output; use `file_path`. **Depends on a crawl-time setting and nothing in the crawl listing reveals it:** on a crawl run without Store HTML it throws `IOException: Not configured to store page content`. Don't ask for a re-crawl on that basis — `Links: All Outlinks` with `Link Path` and `Link Position` gives per-anchor destination, anchor text and the emitting XPath, which is what element-level work usually needed, and it is populated regardless. |
| ⬜ | `sf_export_embeddings` | Exports embeddings, if the crawl generated them. |

**`segment_name`** is unexercised. If the crawl has segments configured it
should scope an export; omitted, the server states it exports all segments.

## Single-URL lookups

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_url_info` | Flat JSON of every stored field for one URL — status, indexability and its reason, inlinks/unique inlinks, crawl depth, word count, timestamps. Excellent for confirming one specific claim cheaply. Values come back as strings, including numerics. `Inlinks` / `Unique Inlinks` **count the page's own self-links** (breadcrumb, share, language-switcher anchors pointing at the page itself) — subtract self before reading the count as "other pages linking here" (see `gotchas.md`). |
| ✅ | `sf_url_links` | Inlinks / outlinks / unique outlinks for a URL, filtered by `links_category` (Hyperlink, HTML Canonical, HTTP Redirect, HTML Hreflang, XML Sitemap, …). Supports `file_path`. **The file is a plain text list of one destination URL per line** — not NDJSON, and with no Anchor, `Link Path` or `Link Position` column, whatever extension you request. For link metadata use the inlink/outlink bulk exports instead. The impoverished output still answers more than it looks: destinations are listed **once per link occurrence**, so counting duplicates gives per-target link frequency from a single page (5× one URL against 1× each parameterised variant) — enough to settle "does this page link X outside component Y" with no DOM access. |
| ⬜ | `sf_url_content` | Stored content for a URL; `show_visible_content_for_html_url` toggles rendered text vs raw HTML. Note this is the **crawl's stored copy**, not a live fetch. |
| ⬜ | `sf_get_url_screenshot` | Screenshot, where the crawl captured one. |
| ⬜ | `sf_open_url_in_browser` | Opens a URL in the user's browser. Side-effecting on their desktop — don't call it incidentally. |

A blocked-by-robots URL has no content, title or word count, because the
crawler never fetched it. `sf_url_info` still works and is the fastest way to
confirm a blocking claim: look for `Status: Blocked by robots.txt` and
`Indexability: Non-Indexable` alongside a real `Inlinks` count.

## Files and execution on the user's machine

These operate on a **fixed base directory on the user's machine**. Where your
shell cannot read it (an agent on another machine), they are the only bridge
between the crawl data and processing. The confinement is a property of the file tools,
not of executed code: a script run with `sf_run_node_js_script` can write
outside the base directory (see `gotchas.md`).

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_list_allowed_base_directory` | Returns the one writable root. Call it once at the start of any file work. |
| ✅ | `sf_list_directories` | `[FILE]`/`[DIR]` listing. `.` lists the base directory. |
| ✅ | `sf_create_directory` | Creates nested paths, succeeds silently if present. **Needed before any `file_path` write into a new folder** — writes do not create parents. |
| ✅ | `sf_write_text_file` | Overwrites without warning. This is how you get a Node script onto the machine. |
| ⬜ | `sf_read_text_file` | Reads back, with `limit` and `skip` for line ranges. Prefer aggregating and returning a summary over reading raw rows. |
| ✅ | `sf_run_node_js_script` | Runs a script **that must already exist on disk** — write it first. `args` accept only `[A-Za-z0-9._]` — a hyphen, slash, space or URL fails before the script runs (`IllegalArgumentException: Argument contains illegal characters`), so pass keys and let the script map each key to its paths and filenames (or read a small config written with `sf_write_text_file`); confirmed at last verification. Stdout comes back as the tool result, so make the script print only what you want to read. **It is plain Node with no injected crawl object** — nothing resembling an `sf` API is available, so a `module.exports = main(sf)` style script produces no output rather than an error. Crawl data reaches a script only through files the export tools wrote. **It is also gated by the server-wide loaded-crawl precondition**, even for work that touches no crawl data: with nothing loaded it fails with `IOException: You need to load or run a crawl first`. Any crawl satisfies the gate, including a one-URL throwaway — keep one loadable for pure file work. The script's filesystem reach is **not** confined to the allowed base directory — unlike the file tools, it can write elsewhere on the machine, which is the cheapest route for moving bulk output into a shared location the session can read directly. **A child process it spawns detached (`spawn` + `unref`) outlives the tool call**, so long-running work is not capped by the call's time budget — the launcher returns immediately and progress is polled with `sf_read_text_file`. That makes this a general execution surface on the user's machine, not just an aggregation helper — including the SEO Spider's own CLI launcher, which a detached spawn with `--crawl-list <file> --headless` turns into the List-mode crawl the tool surface lacks (separate process; the MCP's loaded crawl is untouched; factory defaults unless `--config` names a saved profile): see "Re-checking a list of URLs" in `workflows.md`. |
| ⬜ | `sf_npm_install` | Installs packages for those scripts. Node built-ins (`fs`, `path`) are enough for aggregation work, so this is rarely needed. |

Keep working files in a dedicated subdirectory (e.g. `wk/`) rather than
scattering them in the base directory — it belongs to the user, and
`sf_write_text_file` overwrites silently.

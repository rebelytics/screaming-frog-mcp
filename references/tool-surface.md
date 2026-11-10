# Tool surface

Reference file for the `screaming-frog-mcp` skill (CC BY 4.0 — Eoghan Henn and contributors). Every tool the server exposes on SEO Spider 25, grouped by the job it does. Names are written
without the server prefix — match by suffix.

Load triggers: choosing a tool, or checking whether a capability exists before concluding it does not; starting a crawl with `sf_crawl` or `sf_crawl_from_list` (crawl mode, authentication, config provenance, bounding it); reading or changing the crawl configuration, custom extraction or segments; moving bulk output outside the allowed base directory; deleting or cleaning up stored crawls

**Verification key.** ✅ = exercised directly on 25.1 and behaves as described
here. ◑ = exercised on 24.x under the same name and not re-run since; the
notes are the 24.x observations, so re-verify before relying on them. ⬜ =
present in the tool list but not yet exercised; the description is the
server's own, and its real behaviour, response shape and failure modes are
unknown. Do not present ◑ or ⬜ behaviour as fact — try it, then report back
so this file can absorb the result.

## Contents

- Names that changed with 25
- Crawl lifecycle
  - Crawl mode and list crawls
  - Authentication for MCP-started crawls
  - config_path provenance
  - Bounding an MCP-started crawl
  - Deleting stored crawls
- Background tasks
- Configuration
  - Custom extraction and segments
- Discovery — what can I ask for?
- Bulk data out
- Single-URL lookups
- Files and execution on the user's machine
- Help and external APIs

---

## Names that changed with 25

A client still holding the 24.x tool list calls names the server no longer
has; each returns `Unknown tool` and drops the loaded crawl (see
`gotchas.md`). The 25 names:

| 24.x name | 25 name |
|---|---|
| `sf_list_available_reports` | `sf_list_reports` |
| `sf_list_available_bulk_exports` | `sf_list_bulk_exports` |
| `sf_list_available_filters_for_seo_element` | `sf_list_filters_for_seo_element` |
| `sf_list_available_data_fields_for_seo_element_and_filter` | `sf_list_fields_for_seo_element_filter` |
| `sf_list_allowed_base_directory` | `sf_allowed_base_directory` |
| `sf_run_node_js_script` | `sf_run_node_script` |

Fields changed too: `sf_list_crawls` returns `crawlId` (was
`instanceDirName`), and `sf_crawl_progress` returns `crawlId` (was
`databaseId`).

## Crawl lifecycle

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_list_crawls` | `limit` defaults to 10. Returns `crawlId` (the value `sf_load_crawl` takes), `url`, `spiderMode` (`Spider`\|`List`), `crawlName`, `urlsCrawled`, `percentageComplete`, `time`, `projectVersion`. `percentageComplete` is genuine progress — a low value means the crawl really is partial, **not** that it is unusable (see "Session-start protocol" in SKILL.md). `projectVersion` is the version that last opened the crawl, not the one that ran it (see `gotchas.md`). |
| ✅ | `sf_load_crawl` | Takes `crawl_id` = the `crawlId` value. **A background task:** the call returns at once and `sf_task_status` carries the outcome (`The crawl has been loaded successfully`). Required before any query tool. Loads over an already loaded crawl without a clear. Loaded state is **lost when the host app restarts**. **A crawl stored by an older version is upgraded in place on load** (backup phase, then a database conversion; about 70 s for a 10k-URL crawl) — see `gotchas.md`. ◑ On 24.x it refused with `IllegalStateException: UNEXPECTED_MESSAGE loadCrawl in state: SpiderPausedState` when the app held a paused crawl, and failed with `Crawl is open in another SEO Spider instance` when the crawl was open in the desktop UI; call `sf_crawl_progress` first either way. |
| ✅ | `sf_crawl` | Starts a spider crawl. Takes `crawl_url` (a single start URL), optional `config_path`, `auth_config_path`, `crawl_name`, `project_name`. **Refused in LIST mode** (see [Crawl mode and list crawls](#crawl-mode-and-list-crawls)). Returns `The crawl has successfully started, crawlId: …` as soon as the crawl starts — not a task, so poll `sf_crawl_progress`. Omit `config_path` and the crawl runs under whatever configuration the instance currently holds — read it first (see [Configuration](#configuration) and [Bounding an MCP-started crawl](#bounding-an-mcp-started-crawl)). Starting a crawl evicts the currently loaded one. While it runs, the export, report, lookup, configuration and script tools are refused as busy (see `gotchas.md`). Do not fire this speculatively on a large site. |
| ✅ | `sf_crawl_from_list` | Starts a List-mode crawl from `list_mode_urls` (inline, one URL per line) or `list_mode_file`; same optional arguments as `sf_crawl`. **Refused in SPIDER mode.** Returns `The list mode crawl has successfully started with N URLs, crawlId: …`. **`list_mode_file` needs an absolute path** — a path relative to the allowed base directory fails (see `gotchas.md`). Starts over a loaded crawl without a clear. |
| ✅ | `sf_set_crawl_mode` | `SPIDER` or `LIST`. Switching is **refused while a crawl is loaded** (`Switching mode clears the loaded crawl. Call sf_clear_crawl first, then send this again.`); asking for the mode already active just answers `Already in <MODE> mode`. Either reply lists the settings kept per mode. |
| ✅ | `sf_crawl_progress` | Returns `crawlId` (which crawl is in memory), `crawlProgress` (`completed`, `active`, `waiting`, `percentComplete`), `apiProgress`, `postCrawlAnalysisProgress`, `stateName`, `crawlRunning`. Poll it after `sf_crawl` until `crawlRunning` is false and `postCrawlAnalysisProgress.percentComplete` is 100; `crawlProgress.percentComplete` at 100 is not the end (more can still be queued). **Call this before `sf_load_crawl`** — it tells you both what is loaded and how much of the site is covered — and **before any absence-based conclusion**: `waiting` is the queue (see "Absence and subtraction traps" in `workflows.md`). **Refused while a background task runs** (see [Background tasks](#background-tasks)), so it is not unconditionally answerable. **Instance-scoped:** it describes the crawl held by the MCP's own instance, not one running in the desktop UI (◑ observed on 24.x: while a UI crawl ran it kept reporting the MCP's previous crawl). For crawls started outside the MCP, poll `sf_list_crawls` and judge by `urlsCrawled`. |
| ◑ | `sf_pause_crawl` / `sf_resume_crawl` | Pause and resume. On 24.x: **pausing does not release the instance** — `sf_load_crawl` on a paused crawl failed with the `SpiderPausedState` exception, so pause-load-resume is not a handover route (non-destructive: resuming restored the crawl with progress intact). Pausing was the only way to read from an MCP-started crawl in flight: pause → query → resume behaved cleanly on every repetition. Resume is the state-changing call to fear on a shared instance — it starts fetching the queue under any other session reading the paused crawl (see "Sharing one instance between sessions" in SKILL.md). |
| ✅ | `sf_clear_crawl` | Empties the instance's memory (`Crawl has been cleared`). Works on an idle, completed crawl as well as a paused one, whatever its description says. **On a database-stored crawl it does not delete the project** — the crawl stays in `sf_list_crawls` and reloads. It is the step `sf_set_crawl_mode` asks for, and the release path for handing a paused crawl back to the desktop UI. Confirm the crawl is a stored project before clearing, and re-check `sf_list_crawls` after. On a crawl with no durable home it is destructive. The configuration survives a clear. No tool deletes a stored crawl — see [Deleting stored crawls](#deleting-stored-crawls). |
| ⬜ | `sf_export_crawl` | Exports the crawl itself (as distinct from a report). |

### Crawl mode and list crawls

The instance is in one of two modes, and each crawl tool is refused in the
other with a message that names the fix: `Can't run tool: sf_crawl as the SEO
Spider is in LIST mode. Call sf_set_crawl_mode with mode SPIDER first, then
set any config for this crawl, then send this again.` Nothing starts and a
loaded crawl is untouched.

- **The mode follows the loaded crawl.** Loading a stored List crawl puts the
  instance in LIST mode. Read the mode from an `sf_set_crawl_mode` reply, not
  from memory of the last switch.
- **Eight settings are kept per mode** and swap on a switch:
  `crawlConfig.limitSearchDepth`, `searchDepthLimit`, `robotsTxtMode`,
  `checkLinksOutsideFolder`, `crawlSitemaps`, `autoDiscoverSitemaps`,
  `internalUrlConfig.searchAllSubdomains`, `crawlOutsideStartFolder`. Set any
  of them after switching, never before.
- **The order for a list crawl:** `sf_clear_crawl` (if anything is loaded) →
  `sf_set_crawl_mode(LIST)` → configuration → `sf_crawl_from_list` → poll
  `sf_crawl_progress`. Recipe and the fallbacks: "Re-checking a list of URLs"
  in `workflows.md`.

### Authentication for MCP-started crawls

◑ Three routes were tested on 24.x against an HTTP-basic-auth-protected host;
only one survived a crawl started with `sf_crawl`. None has been re-run on 25.

- **Credentials in the start URL (`https://user:pass@host/`) — failed.**
  They authenticate the start URL itself (200, correct title), then the
  crawl dies there: the Spider rewrites every discovered internal link to
  carry the userinfo **lowercased**, mangling any mixed-case password, and
  no URLs are queued.
- **Credentials saved in the app (Configuration > Authentication) — failed.**
  They did not reach a crawl started through the MCP, with or without a
  `config_path`. Signature: start URL `Status Code: 0`, `Indexability
  Status: No Response`, raw `java.lang.NullPointerException` in the Status
  field — the app attempting to raise its interactive auth dialog with no UI
  to raise it in (see `gotchas.md`). Credentials being saved in the app is
  not evidence that an unattended crawl will authenticate.
- **HTTP header in the crawl config — worked. Default to this for any
  auth-protected target.** `Authorization: Basic <base64(user:pass)>` as a
  custom HTTP header in the crawl configuration. On 25 the
  `customHttpHeadersConfig` category is where `sf_set_config_spider` would
  set it (⬜ not exercised); a saved profile passed as `config_path` is the
  24.x route.

⬜ `auth_config_path` on both crawl tools takes the separate authentication
config the 24.x server could not receive. Unexercised: try it where the
header route does not fit, and report back.

### config_path provenance

◑ Observed on 24.x: a config profile inherits the mode the desktop app was in
when it was saved, and nothing in the saved artefact reveals it. A profile
saved in List mode carries Store-without-Crawl settings on the Spider > Crawl
tab (separate **Crawl** and **Store** checkboxes per link type), and passed to
`sf_crawl` it silently yielded a crawl that recorded the start URL's outlinks
but queued none of them — a one-URL crawl with no error. On 25 the mode is
explicit and each crawl tool is refused in the wrong one, but whether a
profile saved in the other mode still carries those settings into the crawl
has not been re-run. When a crawl completes at one URL, read
`summary.totalUrlsEncountered` from `sf_generate_report(category: "Crawl
Overview")` before investigating anything else — `gotchas.md` has the full
diagnostic — and compare the `crawl*` and `store*` pairs in
`sf_get_config_spider(category: "crawlConfig")`.

### Bounding an MCP-started crawl

**Set the bound in the configuration, then confirm it on the first poll.**
`sf_get_config_spider(category: "crawlConfig")` shows `limitCrawlTotal` and
`crawlTotalLimit`; the default observed is on, at five million — a cap in
name only. `sf_set_config_spider` with `{"crawlConfig": {"limitCrawlTotal":
true, "crawlTotalLimit": N}}` before the crawl stopped a spider crawl at
exactly N, and `completed + waiting` already equalled N at the first poll:
the limit caps the queue, so the first poll tells you it applied. The limit
stays in the instance afterwards — put it back (see
[Configuration](#configuration)).

◑ Where the bound has to be a pause instead (a crawl already running, a
target that cannot be expressed as a count), the 24.x observations stand:
**key the pause on the crawl's own counter, not on the clock** — poll
`completed` and pause when it crosses the target; **treat the first
throughput reading as a lower bound** (on a large site, roughly 400 URLs/min
early on and roughly 1,000/min later, so an interval sized on the first
reading overshoots); and **poll from the session that started the crawl**,
because while it runs little else on the instance can be queried.

### Deleting stored crawls

No MCP tool deletes a stored crawl: `sf_clear_crawl` empties the instance
and the project stays in `sf_list_crawls`. A script run through
`sf_run_node_script` can reach the store
(`~/.ScreamingFrogSEOSpider/ProjectInstanceData/<crawl id>/`), but
never delete or modify those folders from a script: the application keeps
a `DO_NOT_MODIFY_THESE_FILES.txt` in the store root and in every crawl
folder, saying crawl data is managed only from the SEO Spider, and each
folder holds lock and key files beside the data, so a hand deletion
bypasses the application's own bookkeeping. The folder name is the
`crawlId`. Both cleanup routes are the
user's: delete crawls in the application's Crawls dialog, or set crawl
retention to delete automatically (a user-side setting, observed set to
never delete). The agent's share:

- Every crawl started through the MCP is stored, test crawls of three URLs
  included, and so was every CLI list run observed on 24.x, with or without
  `--save-crawl` (`workflows.md`, Route A). Do not launch throwaway runs.
- At session end, name every stored crawl the session created, every CLI
  run included, by `crawlName`, `crawlId` and whether it is still
  needed, so the user can delete the rest in one pass.

Stored crawls and their by-products are what fills the disk under a long
CLI crawl; the free-disk guard and its recovery are in `workflows.md`,
Route A.

## Background tasks

Loads and exports run in the background on 25: the call returns `The task
'<name>' has started and is currently running` and says nothing about the
work. The result is fetched separately.

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_task_status` | Plain text: `Task state: IDLE \| RUNNING \| SUCCEEDED \| FAILED`, `Task: <name>`, then `Running for: Ns` with a `Progress:` line, or `Took: Ns` followed by the result. Before any task: `No task has been run yet`. **It reports the last task that started, not the last call** — after a call that failed synchronously it still shows the previous task's result, and two exports of one filter share a task name. Match the output path before reading a count. Result shapes per export family: `gotchas.md`. |
| ✅ | `sf_cancel_task` | Stops the running task: `Task state: FAILED`, `User cancelled`, no file at the output path, the loaded crawl kept. Its first line, `Cancellation requested and the task has stopped.`, is printed even when the task had already finished — the state line underneath is the answer. |

One task at a time, and while it runs the other tools are refused with a
message naming the task, its progress and its runtime — `sf_crawl_progress`
included. `sf_crawl` and `sf_crawl_from_list` are not tasks; `sf_url_info`
and `sf_url_content` answer synchronously.

## Configuration

The crawl configuration is readable and writable over MCP on 25. It is
**instance state**: it persists across `sf_clear_crawl` and into the next
crawl anyone starts, so read the value before changing it and put it back
when the work is done.

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_list_config_categories` | 37 top-level categories with a one-line description each — `crawlConfig`, `performanceConfig`, `userAgentConfig`, `internalUrlConfig`, `excludeManager`, `customExtractionConfig`, `customSearchConfig`, `customJavaScriptConfig`, `customHttpHeadersConfig`, `customRobotsConfig`, `segmentsConfig`, `urlRewritingConfig`, `crawlAnalysisConfig`, the API categories (`ga4Config`, `gscConfig`, `pageSpeedConfig`, …) and the AI provider ones. |
| ✅ | `sf_get_config_sub_schema` | JSON Schema for one category, with long field descriptions. The descriptions carry rules the tool list does not (segment query syntax, which lists are replaced wholesale). |
| ✅ | `sf_get_config_spider` | Current values, keyed by category; pass `category` or get everything. For `crawlConfig` that is about 120 fields — the cheapest way to learn the field names, and the read-back after every set. |
| ✅ | `sf_set_config_spider` | `config` is a JSON string keyed by category; only the fields present change. `Configuration applied successfully`. **A list field is replaced, not merged** — send every entry you want to keep. **Validation is all-or-nothing:** one invalid entry rejects the whole payload (`Config validation failed: …`, naming the entry), and the valid fields beside it are not applied. Accepted while an idle crawl is loaded; the server says it is refused while a crawl runs. |

Confirmed by a crawl: `{"crawlConfig": {"crawlTotalLimit": 60,
"limitCrawlTotal": true, "crawlerMode": "RENDER", "storeOriginalHtml": true,
"storeRenderedHtml": true}}` produced a 60-URL crawl with rendered content
and stored HTML (60 rendered URLs took about seven minutes at default
speed). `crawlerMode` defaults to `STANDARD`; both Store HTML flags default
to off, and they are what `sf_bulk_export_page_content` depends on.

### Custom extraction and segments

- **Custom extraction** (`customExtractionConfig.filters`): each rule takes
  `name`, `extractionType` (`XPATH`, `CSSPATH`, `REGEX`), `expression`,
  `dataType` (`INNER_TEXT`, `INNER_HTML`, `OUTER_HTML`, `FUNCTION_VALUE`).
  Rules run as pages are crawled, so set them before the crawl; changing
  them afterwards does not re-extract. After the crawl
  `sf_list_filters_for_seo_element("Custom Extraction")` lists `All` plus one
  filter per rule, and the export carries a column per match: `<rule> 1`,
  `<rule> 2`.
- **Segments** (`segmentsConfig.segments`): each takes `idx` (3 to 252),
  `name`, `query`, `rgbColour`. Segments are worked out during the crawl, so
  configure them first. **Call `sf_list_segment_query_columns` before
  writing a query** — column names are internal identifiers (`[Address]`,
  `[Word Count]`), not UI headings (`[Title 1]` is rejected), and an unknown
  column rejects the whole configuration. Its `filter` argument is a
  substring match on name or label, so a short term returns a long list.
  `segment_name` on an element export scoped it as expected (60 rows
  unscoped, 43 in the segment), and rows carry a `Segments` field. A
  numeric condition also catches rows where the field is zero for another
  reason — a word-count threshold matches every redirect.

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_list_segment_query_columns` | `column`, `label`, `group` per entry, for the configuration currently loaded. |
| ⬜ | `sf_list_ai_prompt_targets` | Listed by the vendor for the AI prompt configuration. |
| ⬜ | `sf_list_issue_filter_keys` | Listed by the vendor for the issue-override configuration. |

## Discovery — what can I ask for?

Data is organised as **SEO element × filter**, mirroring the desktop UI. The
names are not guessable and discovery is cheap, so resolve rather than assume.
Element and filter names are matched ignoring case and whether words are
joined by a space, an underscore or a hyphen (`response_codes` + `all`
works); the whole name is still needed.

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_list_filters_for_seo_element` | `seo_element_name` is a fixed enum (Internal, External, Response Codes, Canonicals, Directives, Hreflang, Links, Structured Data, Sitemaps, Page Titles, Validation, AI, Accessibility, Social, …). Returns the filters valid for it. For Custom Extraction, Custom Search, Custom JavaScript and AI the filters come from the configuration, so an unconfigured one lists only `All`. |
| ⬜ | `sf_list_fields_for_seo_element_filter` | Returns the field names valid for that element *and* filter. Not run under its 25 name; under its 24.x name it showed that **field sets differ per filter** — the single most common source of avoidable errors. |
| ✅ | `sf_list_reports` | Lists reports for `sf_generate_report`, one colon-delimited name per line. |
| ✅ | `sf_list_bulk_exports` | A JSON array of `{categoryName, type}`; `type` is `SINGLE_FILE` or `MULTI_FILE`. A multi-file export takes a directory as `output_path` and is always CSV. |

Useful filter sets observed:

- **Response Codes** — `Internal Blocked by Robots.txt`, `Internal Success (2xx)`, `Internal Redirection (3xx)`, `Internal Client Error (4xx)`, plus External and non-prefixed variants of each.
- **Canonicals** — `Canonicalised`, `Self Referencing`, `Non-Indexable Canonical`, `Missing`, `Multiple Conflicting`, `Unlinked`, `Outside <head>`.
- **Directives** — `Noindex`, `Index`, `Nofollow`, `NoArchive`, `Unavailable_After`, …
- **Internal** — `HTML`, `JavaScript`, `CSS`, `Images`, `PDF`, …

## Bulk data out

All four exercised export tools are background tasks and **require** a
`file_path` (or `output_path`): omitting it fails with `file_path argument
missing or blank`. The result — receipt, count, saved location — comes from
`sf_task_status`.

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_export_seo_element_urls` | The workhorse. NDJSON. `data_fields` omitted = all fields; also takes `max_rows`, `start_index`, `segment_name`. The task result is the JSON receipt `{format, path, total_records, sample}`, and here `path` is the real file. Three traps: `max_rows` **caps the receipt's `total_records`**, so it cannot be used to probe a count; an **empty filter starts no task** — the call fails with `No URLs match filter '…' of …, so there is nothing to export`; and `urlContent`, exported only when named in `data_fields`, puts page text in every row and in the receipt's `sample`. |
| ✅ | `sf_generate_report` | `category` is colon-delimited (`Category:Subcategory`); supports `CSV`/`NDJSON`. **`category: "Crawl Overview"` is the population cross-check** — it carries totals the filters don't store (`Total External blocked by robots.txt`, `URLs Displayed`, content-type breakdowns), which is how you tell a genuine zero from an unpopulated filter, and `summary.totalUrlsEncountered` — the first thing to read when a crawl completes at one URL (see `gotchas.md`). **Crawl Overview is not NDJSON:** the file is a single pretty-printed JSON document, whatever extension you asked for, and its task result is a prose list of the six objects inside plus `Export saved to …` — no receipt, no count. Tabular reports (`Redirects:All Redirects`) return the JSON receipt with `total_records`, but their `path` is a temp file; the real location is the `Export saved to …` line. |
| ✅ | `sf_generate_bulk_export` | Bulk export equivalent; `output_path`. Single-file exports return the receipt with the same `path` caveat as tabular reports — **read `Export saved to …`, not `path`**. An **empty export ends `SUCCEEDED` with `Report generated no data. No file was written to …`** — apply the same true-zero-vs-unpopulated-filter caveat. A multi-file export returns `… successfully exported` and the directory, no count. `Links:All Inlinks` on a 10k-URL crawl wrote 640k rows in 12 s and left the crawl loaded. ◑ On 24.x a heavy export (`Queued URLs` on a ~1M-row queue, `Links: All Inlinks` on a ~100k-URL crawl) timed out at the MCP layer, wrote nothing and dropped the loaded crawl; not reproduced on 25 and not tested at that size — see `gotchas.md`. |
| ✅ | `sf_bulk_export_page_content` | `page_content_type` is `RAW_HTML`, `VISIBLE_TEXT` or `MARKDOWN`; NDJSON rows of `{Address, content}`. **Depends on Store HTML at crawl time:** on a crawl run without it the call fails with `Not configured to store page content`, and nothing in the crawl listing reveals the setting beforehand. On 25 the setting can be switched on through the configuration tools before the crawl. Don't ask for a re-crawl on a refusal alone — `Links: All Outlinks` with `Link Path` and `Link Position` gives per-anchor destination, anchor text and the emitting XPath, which is what element-level work usually needed. The receipt's `sample` is one whole page. |
| ⬜ | `sf_export_embeddings` | Exports embeddings, if the crawl generated them. |

## Single-URL lookups

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_url_info` | Flat JSON of every stored field for one URL — status, indexability and its reason, inlinks/unique inlinks, crawl depth, word count, timestamps, `Segments`. Synchronous; with `file_path` it answers `Successfully saved to …`. Excellent for confirming one specific claim cheaply. Values come back as strings, including numerics. `Inlinks` / `Unique Inlinks` **count the page's own self-links** (breadcrumb, share, language-switcher anchors pointing at the page itself) — subtract self before reading the count as "other pages linking here" (see `gotchas.md`). |
| ◑ | `sf_url_links` | Inlinks / outlinks / unique outlinks for a URL, filtered by `links_category` (Hyperlink, HTML Canonical, HTTP Redirect, HTML Hreflang, XML Sitemap, …; 25 adds `Open Graph Image` and `X Card Image`). Supports `file_path`. On 24.x **the file was a plain text list of one destination URL per line** — not NDJSON, and with no Anchor, `Link Path` or `Link Position` column, whatever extension was requested. For link metadata use the inlink/outlink bulk exports instead. The impoverished output still answers more than it looks: destinations are listed **once per link occurrence**, so counting duplicates gives per-target link frequency from a single page — enough to settle "does this page link X outside component Y" with no DOM access. |
| ✅ | `sf_url_content` | The crawl's stored copy of one URL, **not a live fetch**. `page_content_type` is `RAW_HTML` (default), `VISIBLE_TEXT` or `MARKDOWN`; the 24.x argument `show_visible_content_for_html_url` is rejected. Synchronous; with `file_path` it answers `Page content successfully saved to …`. An image URL returns base64. |
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
not of executed code: a script run with `sf_run_node_script` can write
outside the base directory (see `gotchas.md`).

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_allowed_base_directory` | Returns the one writable root. Call it once at the start of any file work. |
| ✅ | `sf_list_directories` | `[FILE]`/`[DIR]` listing. `.` lists the base directory. |
| ✅ | `sf_create_directory` | Creates nested paths, succeeds silently if present. Create the folder before the first `file_path` write into it (◑ on 24.x writes did not create parents). |
| ✅ | `sf_write_text_file` | Overwrites without warning. This is how you get a Node script or a URL list onto the machine. Answers during a running crawl. |
| ✅ | `sf_read_text_file` | Reads back, with `limit` and `skip` for line ranges. **A file over 100,000 bytes is rejected, not truncated** (`Limit exceeded … Read it in pages using the 'skip' and 'limit' arguments`). Prefer aggregating and returning a summary over reading raw rows. |
| ✅ | `sf_run_node_script` | Runs a script **that must already exist on disk** — write it first. `args` accept only `[A-Za-z0-9._]` — a hyphen, slash, space or URL fails before the script runs (`Argument contains illegal characters. Only alphanumeric and ._ are allowed.`), so pass keys and let the script map each key to its paths and filenames (or read a small config written with `sf_write_text_file`). Stdout comes back as the tool result, so make the script print only what you want to read. The working directory is the base directory. **It is plain Node with no injected crawl object** — crawl data reaches a script only through files the export tools wrote. **It is gated by the server-wide loaded-crawl precondition**, even for work that touches no crawl data: with nothing loaded it fails with `You need to load or run a crawl first`, and while a crawl runs it is refused as busy. Any loaded crawl satisfies the gate, including a three-URL list crawl — keep one loadable for pure file work. The script's filesystem reach is **not** confined to the allowed base directory, which is the cheapest route for moving bulk output into a shared location the session can read directly. ◑ On 24.x **a child process it spawned detached (`spawn` + `unref`) outlived the tool call**, so long-running work was not capped by the call's time budget — which makes this a general execution surface on the user's machine, including the SEO Spider's own CLI launcher: see "Re-checking a list of URLs" in `workflows.md`. |
| ⬜ | `sf_npm_install` | Installs packages for those scripts. Node built-ins (`fs`, `path`) are enough for aggregation work, so this is rarely needed. |

Keep working files in a dedicated subdirectory (e.g. `wk/`) rather than
scattering them in the base directory — it belongs to the user, and
`sf_write_text_file` overwrites silently.

## Help and external APIs

| | Tool | Notes |
|---|---|---|
| ✅ | `sf_check_faq_before_answering` | The vendor's FAQ: without `ids`, a list of about 70 questions in nine sections, each with an id; with `ids`, short answers. Check it before explaining a failed crawl or a setting from general knowledge. The application downloads the FAQ from the vendor at call time, so it needs network and can time out (`did not respond within 15s`; a retry worked). Answers during a running crawl. It does not report the installed version. |
| ✅ | `sf_list_api_connections` | `googleApis` (GA4, Search Console, each with its saved accounts and selection) and `otherApis` (PageSpeed, Majestic, Ahrefs, Moz and the AI providers), each with `connected`, `hasStoredCredentials`, `hasCredits` and the `configCategory` its options live in. Read-only. **An API's columns come back empty when it is not connected** and the crawl still succeeds, so check here before reading empty API columns as zeros. |
| ⬜ | `sf_connect_api` / `sf_disconnect_api` | Connect or disconnect a provider for the instance. State-changing for every session on it; where `hasStoredCredentials` is false the user has to supply the key in the desktop app first. |

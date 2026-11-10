# Gotchas and boundaries

Reference file for the `screaming-frog-mcp` skill (CC BY 4.0 — Eoghan Henn and contributors). Things that error, mislead, or cannot be done at all. Ordered roughly by how
early in a session they bite.

Load triggers: before the first export (task result shapes); a call errors unexpectedly, or a running crawl or task refuses queries; an empty filter or row export disagrees with a summary count; before a state-changing call on a shared instance

**Version key.** Entries describe SEO Spider 25.1 unless tagged **[24]**: behaviour observed on 24.x and not re-run since the upgrade. A tagged entry is a lead, not a fact — re-verify it before relying on it and report back.

## Contents

- Errors you will hit
- What `sf_task_status` returns, by export family
- Silently misleading values
- Hard boundaries — no workaround
- Side effects on the user's machine — and on other sessions

---

## Errors you will hit

**`Error: Unknown tool: invalid_tool_name` — and the loaded crawl is gone**
The client is calling a name the server does not have: almost always a
client that kept the previous release's tool list across an application
upgrade (the six renames are in `tool-surface.md`). The call does more than
fail: afterwards `sf_crawl_progress` reports `SpiderNoDataIdleState` under a
new `crawlId`, and the next data call fails with `You need to load or run a
crawl first`. Unchanged names keep working, which makes it read like a
half-broken server. Treat a failure on a name that used to work as a version
question first: compare the tool names the session holds with the fields
`sf_list_crawls` returns, and have the user restart the client or open a new
session — a new session picked up the 25 list. Do not probe with further old
names; each one costs the loaded crawl. Whether the drop is a server restart
or a state reset is not established.

**`A task is currently running for <name>. Progress: … Running for Ns.`**
A background task (a load or an export) is in flight and every other tool
is refused until it ends — `sf_crawl_progress` included. Poll
`sf_task_status`, or stop the task with `sf_cancel_task`. One task at a
time.

**`No URLs match filter '<filter>' of <element>, so there is nothing to export.`**
The filter holds zero rows; no task is started and no file written. Read it
with the two-cause caveat under "Silently misleading values" — and note
that `sf_task_status` still shows the task before it.

**`Can't run tool: sf_crawl as the SEO Spider is in LIST mode` (and the mirror for `sf_crawl_from_list`)**
Each crawl tool runs in one mode only. The message names the fix
(`sf_set_crawl_mode`); nothing was started. If a crawl is loaded the switch
itself is refused with `Switching mode clears the loaded crawl. Call
sf_clear_crawl first, then send this again.` Loading a stored List crawl
puts the instance in LIST mode, so the mode can change without anyone
switching it.

**`The SEO Spider hit an internal error (NoSuchFileException): <relative path>`**
`sf_crawl_from_list` was given a `list_mode_file` relative to the allowed
base directory. Unlike the file tools, this argument is not resolved against
it: pass the absolute path. The message adds that sending different
arguments will not help, which is wrong for this cause. A loaded crawl
survives the error.

**`Config validation failed: <category>.<field>…`**
`sf_set_config_spider` rejected the payload, and rejected all of it: the
valid fields beside the failing one were not applied. The message names the
entry and usually the tool that lists valid values (an unknown segment
column points at `sf_list_segment_query_columns`). Fix the one entry and
resend the whole payload.

**`file_path argument missing or blank`**
Exports no longer return rows inline; every export needs a path to write to.

**`Limit exceeded: this file is larger than the maximum response size of 100000 bytes`**
`sf_read_text_file` rejects a file over the limit instead of truncating it.
Page through it with `skip` and `limit`, or aggregate it in a script.

**[24] `IllegalStateException: UNEXPECTED_MESSAGE loadCrawl in state: SpiderPausedState`**
The desktop app is holding a paused crawl and will not load another over it.
(On 25 a load over an idle, completed crawl is accepted; the paused case has
not been re-run.) Do **not** resume it to make the load succeed — that
discards someone's in-progress work. Call `sf_crawl_progress`: it reports
the `crawlId` in memory, and very often that is already the crawl you wanted, in which case
query it in place and skip the load entirely.

**[24] Pausing a running crawl does not help either** — the same exception is what
you get after pausing, so pause-load-resume is not a handover route. Resuming
afterwards did restore the crawl with progress intact, so the attempt is
non-destructive, but it buys nothing.

**When you genuinely need to free the instance, `sf_clear_crawl` is the
release path — with a storage-model qualifier.** On a database-stored crawl it
clears only the instance's memory: the project stayed in `sf_list_crawls` with
all 33,387 URLs and reloaded immediately, so the user can pick it up in the
desktop UI. Confirm the crawl is a stored project first, and re-check
`sf_list_crawls` afterwards rather than trusting the model. An operation that
discards in-memory state is only dangerous when the state has no durable home
— but establish that it has one before relying on it.

**[24] `Crawl is open in another SEO Spider instance`**
The crawl you asked `sf_load_crawl` for is open in the desktop UI. A crawl
run or opened in the UI stays locked to that instance, and the MCP cannot
load it until the user closes it there. This recurs on every UI→MCP
handover, so state it up front — "close the crawl in the app, then I'll
load it" — rather than rediscovering it per attempt.

**`You need to load or run a crawl first`**
No crawl is loaded. Run `sf_load_crawl`. The non-obvious case: this appears
*mid-session* after the user restarts the host application, on tools that
worked minutes earlier. Loaded state is not durable. Re-load rather than
diagnosing a broken server.

Two more paths into it. **The precondition is server-wide, not per-tool:
`sf_run_node_script` raises it too**, even for a script that only reads and
aggregates files on disk and touches no crawl data at all. Loading any crawl
satisfies the gate — a three-URL list crawl is enough — so keep one loadable
when doing pure file work. And **a call to an unknown tool name drops the
loaded crawl** (first entry above), so this error frequently arrives as the
*second* symptom of a failure whose first symptom looked unrelated: re-load
before concluding anything about the next call.

**[24] A heavy export that times out drops the loaded crawl — for every
session on the instance**
On 24.x exports ran inside the call, and a large one timed out at the MCP
layer: `Queued URLs` on a ~1M-URL list-crawl queue (twice) and `Links: All
Inlinks` on a ~100k-URL crawl. The work aborted with the request — no
partial file — and the loaded crawl went with it: `sf_crawl_progress`
reported `SpiderNoDataIdleState` under a different id, and any other
session's next call failed with `You need to load or run a crawl first`.
The size was what mattered, not the category.

On 25 exports are background tasks, so there is no call left to time out,
and the one heavy export tried — `Links:All Inlinks` on a 10k-URL crawl,
about 640k rows — finished in 12 s with the crawl still loaded. That is
not yet evidence about an export that runs for minutes. Until one has been
observed: read `waiting` and `urlsCrawled` first to size the export, poll
`sf_task_status` instead of waiting on it, cancel with `sf_cancel_task`
when it outgrows its purpose, and check `sf_crawl_progress` afterwards
before the next data call (see "Sharing one instance between sessions" in
SKILL.md).

**`Can't run tool: <tool> as the SEO Spider is busy in SpiderActiveState`**
A crawl is running in the instance (typically one you started), and the
exports, reports, URL lookups and `sf_run_node_script` are refused until
`crawlRunning` is false — the script runner too, even for file-only work.
`sf_crawl_progress`, `sf_check_faq_before_answering`,
`sf_list_api_connections` and `sf_write_text_file` still answered. **[24]**
To read from a crawl in flight the loop was `sf_pause_crawl` → query →
`sf_resume_crawl`, which behaved cleanly on every repetition. This is not
the `SpiderPausedState` refusal above: that one blocks *loading* over a
paused crawl, this one blocks *reading* from an active crawl, and pausing
fixes only the second.

**[24] `IOException: The following name/s are invalid ["…"]. The valid names you can use are: […]`**
Data-field names are **per element *and* filter**, not global. `Unique
Inlinks` is valid on `Internal`/`HTML` and invalid on `Response Codes`/
`Internal Blocked by Robots.txt`. The error enumerates the valid set, so this
costs one round trip rather than a wrong answer — but
`sf_list_fields_for_seo_element_filter` avoids it. (On 25 an unknown
*argument* is rejected the same way, with the accepted list, and the loaded
crawl survives it.)

**`Not configured to store page content`**
`sf_bulk_export_page_content` on a crawl run without Store HTML. The setting
was chosen at crawl time and nothing in `sf_list_crawls` reveals it
beforehand, so this is discovered by trying. For a crawl you are about to
start, switch `storeOriginalHtml` / `storeRenderedHtml` on through
`sf_set_config_spider` first (`tool-surface.md`, Configuration). It is not a reason to ask for a
re-crawl: check what the crawl *does* hold first. `Links: All Outlinks` with
the `Link Path` and `Link Position` fields recovers per-anchor destination,
anchor text and the emitting XPath — the element-level attribution the
content export was usually wanted for — on the same crawl that refused the
export.

**[24] `NoSuchFileException: /…/wk/out.ndjson`**
`file_path` does **not** create parent directories. Call
`sf_create_directory` first. The path in the error is the absolute path on
the user's machine, which is also a quick way to confirm the base directory.

**[24] `Status Code: 0` on the start URL, `Indexability Status: No Response`, and
a raw `java.lang.NullPointerException` in the Status field**
An MCP-started crawl hit an auth-protected target whose credentials were
saved in the app under Configuration > Authentication. Those credentials do
not reach a crawl started through the MCP: the app tries to raise its
interactive authentication dialog with no UI context to raise it in, and
fails as an internal error rather than as "interaction required". The
signature reads like a broken MCP or a broken app; the diagnosis that
actually leads somewhere is "this crawl needed a dialog nobody could
answer". The same failure occurs with and without `config_path`, so it is
not a property of the saved profile. The method that works is HTTP-header
auth — see "Authentication for MCP-started crawls" in `tool-surface.md`.

---

## What `sf_task_status` returns, by export family

Plain text: `Task state: IDLE | RUNNING | SUCCEEDED | FAILED`, `Task:
<name>`, then `Running for: Ns` and a `Progress:` line, or `Took: Ns` and
the result. The result differs by tool:

| Export | Result | Where the file is | Count |
|---|---|---|---|
| `sf_export_seo_element_urls` | JSON receipt `{format, path, total_records, sample}` | `path` | `total_records` |
| `sf_generate_report`, tabular (e.g. `Redirects:All Redirects`) | the same receipt, then `Export saved to …` | the `Export saved to` line (`path` is a temp file) | `total_records` |
| `sf_generate_report`, `Crawl Overview` | prose naming the six objects in the document, then `Export saved to …` | the `Export saved to` line | none |
| `sf_generate_bulk_export`, single file | the receipt, then `Export saved to …` | the `Export saved to` line (`path` is a temp file) | `total_records` |
| `sf_generate_bulk_export`, multi-file | `… successfully exported`, then `Export saved to <dir>` | the directory | none |
| `sf_bulk_export_page_content` | the receipt; `sample` is one whole page | `path` | `total_records` |
| `sf_load_crawl` | `The crawl has been loaded successfully` | — | — |
| any, cancelled | `Task state: FAILED`, `User cancelled` | nothing written | — |
| bulk export, empty | `Task state: SUCCEEDED`, `Report generated no data. No file was written to …` | nothing written | — |

`sf_url_info` and `sf_url_content` are synchronous and never appear here;
neither do `sf_crawl` and `sf_crawl_from_list`, which are polled through
`sf_crawl_progress`.

---

## Silently misleading values

These produce no error, which is what makes them worth memorising.

**A low `percentageComplete` is accurate, and is not a reason to reject the
crawl.** The value is real progress — a crawl reading `7.0` genuinely has most
of the site uncrawled, and `sf_crawl_progress` will show the queue behind it
(62,551 completed against 785,256 waiting, in one case). The error is the
inference rather than the reading: partial and paused crawls are usually
exactly what the user means by "the last crawl", because large sites are
sampled deliberately. Treat the number as telling you how to caveat the
counts — they are floors — not whether the crawl is usable. Component-level
findings from a sample generalise; totals do not.

**A `percentageComplete` of `100.0` is a rounded ratio, not an empty
queue.** A paused ~100k-URL crawl with a hundred-odd URLs still waiting
lists as `100.0` in `sf_list_crawls`; only `sf_crawl_progress` (`waiting`,
`stateName: SpiderPausedState`) shows the remainder. The remainder is not
noise: exports hold fetched URLs, the queue holds discovered-but-unfetched
ones, and the queue is the tail of discovery, so it concentrates the
late-found, lightly-linked URLs an orphan or "not linked" analysis is
looking for. In one such analysis every "orphan" a sitemap diff produced
was sitting in the queue — discovered via a link, which is the opposite of
an orphan. Guard and recipe: "Absence and subtraction traps" in
`workflows.md`.

**`Inlinks` and `Unique Inlinks` include the page's own self-links.** A
template that links to itself — breadcrumb tail, share links, a language
switcher whose current-language entry is an anchor — adds a phantom source
to every page it renders, uniformly, so the shift never looks like an
outlier. Any "sources from class X minus sources from class Y" subtraction
that treats the remainder as real inbound links is off by one per page,
and a threshold near zero reads exactly wrong: "one non-X source" is
usually "zero plus self". Subtract self first (`sf_url_links`, inlinks,
`Hyperlink`, drop rows whose source equals the page), and before trusting
any aggregate built on inlink counts, pull the actual sources of the single
weakest item the method produced and classify them by hand. See "Absence
and subtraction traps" in `workflows.md`.

**[24] `sf_crawl_progress` is scoped to the MCP's own instance and does not
name the crawl a user is watching.** While a crawl the user started in the desktop UI
is running, the progress tool keeps reporting whatever the MCP's instance
last held — e.g. `completed: 283, percentComplete: 100` while the user's
crawl is minutes old and growing. Read naively, that says "your crawl is
done" about an entirely different job — a confident status about the wrong
one. For any crawl started outside the MCP's instance, poll `sf_list_crawls`
and judge progress by `urlsCrawled` on the new crawl's id.

**A crawl that finishes at one URL with no error has two different causes,
and per-URL data does not distinguish them.** Read
`summary.totalUrlsEncountered` from
`sf_generate_report(category: "Crawl Overview")` **before investigating
anything else**. Encountered 1 with no outlinks and a non-200 start URL
means the page failed. Encountered 1 with outlinks present and a 200 means
links were stored but never queued — the crawl recorded what it found and
acted on none of it. The known cause of the second **[24]**: the `config_path`
profile was saved while the desktop app was in List mode. The Spider > Crawl
tab holds separate **Crawl** and **Store** checkboxes per link type, and a
Store-without-Crawl state produces exactly this signature, so a List-mode
profile silently yielded a one-URL spider crawl. On 25 the pairs are
readable as `crawl*` / `store*` fields in `sf_get_config_spider`.
The outlink count on the start URL looks healthy in both the working and the
broken case (123 recorded outlinks in the broken one), so it cannot settle
the question. After re-saving the profile in Spider mode, the same call
climbed past 6,800 URLs. See "config_path provenance" in `tool-surface.md`.

**`sf_task_status` reports the last task that started, not the last call.**
A call that fails synchronously — an empty filter, a missing argument, a
refusal — starts no task, so the status afterwards is still the previous
export's result, count and sample included. Two exports of the same filter
also share a task name. Before reading a number off a status, check that
the path in it is the file you just asked for.

**A success state is not a non-empty result, and a cancel line is not a
cancellation.** An empty bulk export ends `SUCCEEDED` with `Report generated
no data`. `sf_cancel_task` prints `Cancellation requested and the task has
stopped.` even when the task had already finished — followed by `Task state:
SUCCEEDED` and the full receipt, with the file on disk. Read the state line
and the text under it, never the first line.

**A file that exists is not a finished file.** A running task's output file
can be read before the task ends. Completion is `sf_task_status`, not the
file's presence.

**The receipt's `sample` can be a whole page.** It is one full row, so a
page-content export, or any export with `urlContent` among its fields,
returns several kilobytes of page text in the status result. Nothing
suppresses it; expect it when polling.

**`projectVersion` is the version that last opened a crawl, not the one that
ran it.** Loading a crawl stored by an older release upgrades it in place (a
backup phase, then a database conversion) and the listing shows the new
version afterwards. So the field cannot date a crawl's data, and the load
itself changes the stored crawl — whether the older application can still
open it afterwards was not tested.

**The configuration is instance state, and so is the mode.** A URL limit, a
rendering mode, a custom-extraction list or a segment set applied with
`sf_set_config_spider` stays in the instance after `sf_clear_crawl` and
applies to the next crawl anyone starts; the crawl mode follows whichever
crawl was loaded last. A crawl that stops at a round number, renders when
nobody asked it to, or is refused for its mode is usually a setting left
behind. Read before a crawl (`sf_get_config_spider`), restore after.

**A list field in the configuration is replaced by what you send.** Sending
one new custom-extraction rule or one new segment deletes the others. Read
the list, add to it, send it whole.

**File output format is a property of the tool, not of the server — and the
extension you asked for is not evidence of what is inside.** Three families,
three formats, all writable to a path you named `.ndjson`:

- `sf_export_seo_element_urls` and the bulk exports — genuine NDJSON, one
  JSON object per line.
- `sf_generate_report` — **one pretty-printed JSON document** spanning many
  lines. The standard `split('\n').map(JSON.parse)` idiom fails on line 1
  with a SyntaxError; parse the whole file at once. This collides with the
  mandated Crawl Overview cross-check, so it recurs.
- `sf_url_links` — a **plain list of URLs**, one per line, with no JSON and
  no link metadata at all.

Sniff before you parse: read a few lines back with `sf_read_text_file`, or
branch in the script on the first non-whitespace character. When an output
looks impoverished, check what its structure still answers before switching
tools — `sf_url_links` lists a destination once per link *occurrence*, so
duplicate counting yields per-target link frequency from a single page
without any metadata columns.

**`max_rows` caps `total_records` in the export receipt.** `max_rows: 1`
against a 2,409-row filter reports `total_records: 1`. No error, and it looks
like a count. Never use `max_rows` to probe how many records exist — export
uncapped and read the receipt.

**An empty filter fails the call — and "empty" has two very different
causes.** `No URLs match filter '…' of …, so there is nothing to export`
means the filter holds zero rows. The message adds that "the selection is
empty rather than the request being wrong", which is true of the request
and says nothing about the site. It is **not** the same as "zero exist".
Either:

- there genuinely are none; or
- **the crawl's configuration never collected or never stored them.**
  Screaming Frog's "Show external URLs blocked by robots.txt" is off by
  default: the crawl *counts* those URLs but does not keep them, so the filter
  is empty while the true figure is not. Equally, if resource crawling or JS
  rendering was off, the CSS/JS/image filters are empty by construction.

**Sibling tools signal "empty" through different channels — a bulk export
succeeds.** `sf_generate_bulk_export` on an empty report runs as a task that
ends `SUCCEEDED` with `Report generated no data. No file was written to …`,
where `sf_export_seo_element_urls` fails the call for the same condition.
Never treat a `SUCCEEDED` state, or the absence of an error, as data
present: read the result text before assuming a file exists. The same
two-cause caveat applies — "no data" means a true zero OR an unpopulated
filter; cross-check against Crawl Overview exactly as below. Treat both
empty signatures identically: as *unproven*, never as zero.

Two harder lessons from a 55k-URL verification session **[24]**:

- **When a bulk export is empty, re-derive the same population through a
  per-filter inlink export before concluding anything.** The two paths are
  not equivalent: the non-indexable-page-inlinks bulk export returned
  `Report generated no data` while the per-filter inlink exports for the
  same underlying rows returned 140 and 1,451 records.
- **The Crawl Overview summary counts independently of the display layer,
  so a non-zero summary beside an empty row export is not a contradiction
  to debug.** The server reported 787 external-blocked-by-robots URLs in
  the summary while both the bulk export and the row-level filtered export
  for the same population came back empty, because a crawl-configuration setting
  governs whether that URL class is *displayed* at all — the summary counts
  it regardless. A disagreement between the two surfaces locates a
  configuration boundary, not a bug, and **the summary is the count to
  trust**.

**Report and bulk-export receipts: read the "Export saved to …" line, not
the `path` field.** For tabular reports and single-file bulk exports the
receipt's `path` points to an internal temp file
(`.ScreamingFrogSEOSpider/temp/tmpCrawlOverviewReport`); the saved location
is only in the trailing "Export saved to …" text. For element exports and
the page-content export `path` is the real file. A script fed the wrong one
processes the wrong file without an error — the table above has each case.

**Prove the filter could have been populated before you report zero.**
`sf_generate_report(category: "Crawl Overview")` carries totals the filters
don't. In one crawl it reported `Total External blocked by robots.txt: 1,158`
next to `URLs Displayed: 61,393` — exactly 1,158 short of the 62,551
encountered — while the matching filter raised the empty-set exception. Also
check the parent population: a crawl holding 0 CSS and 0 JS URLs cannot report
a blocked script whatever the truth.

Reporting an unpopulated filter as a clean zero is the worst outcome available
here, because it reads as a verified all-clear rather than as missing data.

**The `Rel` column in inlink/outlink exports is populated for some
relations and empty for others — check before filtering on it.** Two
observations that do not yet reconcile. On a multi-domain crawl **[24]**,
`Rel` came back `null` on every link row, country-switcher anchors included,
though they verifiably carried `rel="alternate"` in the page source. On a
crawl run by 25.1, Hyperlink rows carried anchor values that are not
directives — `noopener`, `noopener noreferrer`, `ugc` — beside `nofollow`,
while every link element in the head (`HTML Canonical`, `HTML Hreflang`)
and every resource row showed `null`. Whether the release changed this, or
`rel="alternate"` on an anchor is a case of its own, has not been tested.
So never build a filter on `Rel` without first counting its values across
a population known to carry the relation you want: a filter that returns
nothing reads, in a link-counting task, as "the component emits no links".
Classify the emitter with **`Link Position`** (Navigation for switcher
anchors vs Head for hreflang link elements) and pin the component with
**`Link Path`**, cross-checking the two. The general rule: a column
existing in an export schema is not evidence it is populated for the
values you care about, and a claim that a field is never populated for a
class of values should name which values were seen empty and on which
release — a field that is null where you know the attribute exists fails
silently as absence rather than loudly as error.

**`sf_url_info` returns everything as strings**, including counts and status
codes. Coerce before comparing or sorting.

**Blocked and unfetched URLs return zeroed fields.** Word count, size, title
and outlinks all come back as `0`/empty for a URL the crawler never fetched.
That is absence of data, not a finding about the page. The fields that remain
meaningful are `Inlinks`, `Status`, `Indexability` and `Crawl Depth`.

**A canonical on a robots-blocked URL cannot be observed.** The crawler never
fetched the page, so it cannot read the tag. Such URLs surface under
*blocked*, never under *canonicalised* — which means an empty "Canonicalised"
report is **not** evidence that canonicals are fine. It may mean the
canonicalised URLs are all blocked. State this limit explicitly rather than
reporting a clean result.

**Crawl counts are a floor.** Depth- and count-limited crawls under-report
absolute volumes. Component-level findings still generalise; totals don't.
Say which is which.

**The crawl's headline total is a union of internal and external URLs —
resolve the denominator before quoting any rate against it.** A ~55k "URLs
crawled" total contained only ~15k pages of the site under discussion; the
rest were external destinations and images. A defect rate quoted "out of
~55k crawled URLs" understated the real rate by ~3.6x and invited the
reader to picture 55k site pages having been checked. Nothing in the
number is wrong — both denominators are defensible in isolation — which is
what makes it dangerous: the reader silently substitutes the population
the sentence implies. Before any "out of N" figure reaches a stakeholder,
name the population N actually covers (internal vs external, crawled vs
encountered, links vs distinct destinations, URLs vs pages) and confirm it
matches what the sentence implies; where the crawl is a sample, say so and
give the sampled population, not the tool's headline total.

---

## Hard boundaries — no workaround

**List mode is a tool since 25 — and the CLI is still there behind it.**
`sf_set_crawl_mode(LIST)` and `sf_crawl_from_list` run a List-mode crawl in
the MCP's own instance (`tool-surface.md`, "Crawl mode and list crawls";
recipe in `workflows.md`, "Re-checking a list of URLs"). The 24.x route —
`sf_run_node_script` spawning the SEO Spider's command-line launcher with
`--crawl-list <file> --headless` — remains the fallback for a list crawl
that must not touch the instance's loaded crawl or its configuration, since
it runs as a separate process. Two rules travel with the CLI route **[24]**.
**A CLI launched from a script inherits factory defaults, not the user's
crawl setup** — no rendering, no auth, no custom extraction — so look for a
saved `.seospiderconfig` profile, saved in List mode, and pass it with
`--config` before reading a failure as a property of the site. And **it is a
separate process**: the MCP's loaded-crawl state is untouched, and the
runner still needs some crawl loaded to launch at all.

Don't improvise a list crawl by starting a spider crawl and stopping it.
For page-level questions on a handful of URLs — meta robots, canonicals,
hreflang, which links a template emits — a browser or HTTP client reads raw
source perfectly well. **But note it will ignore robots.txt**, so it can
confirm what a page contains and cannot confirm whether a URL is blocked.
Blocking is a robots.txt evaluation, and this server already answers it via
`Matched Robots.txt Line`.

**Where your shell cannot see the base directory, files move only through
the server.** It reads and writes one fixed root on the SEO Spider's machine.
From an agent on another machine, use `sf_write_text_file` /
`sf_read_text_file` / `sf_run_node_script`: you cannot hand it a local
script by path or open its exports directly. On the same machine your shell
reads the root directly, but a script that prints only what you need is cheaper.

The confinement applies to the *file tools*, not to code the server
executes: a script run with `sf_run_node_script` can write outside the
allowed base directory. That is the cheapest route for moving bulk output
into a shared location the session can read directly — a zero-context file
handoff. Similarly **[24]**, `sf_crawl`'s `config_path` accepted an absolute
path outside the base directory, and on 25 `list_mode_file` requires one.

**`sf_run_node_script` runs a file, not a string.** Write it first with
`sf_write_text_file`. `args` take only letters, digits, `.` and `_` — no
hyphen, slash, space or URL — so pass keys and let the script map them to
paths.

**Credentials: the header route is the verified one.** On 24.x the server
took one config file per crawl, so the separate authentication config
could not be supplied and app-saved credentials never reached an
MCP-started crawl; HTTP-header auth in the ordinary configuration did. On
25 both crawl tools also take `auth_config_path`, and the header can be set
through `customHttpHeadersConfig` — neither exercised yet (see
"Authentication for MCP-started crawls" in `tool-surface.md`).

**One crawl at a time — per instance, not per session.** Every query runs
against whatever is loaded. There is no per-call crawl scoping, so
comparing two crawls means loading one, exporting, loading the other,
exporting, and diffing outside the server. And the loaded state is shared
by every MCP client driving the instance: a second session cannot hold a
different crawl, and its load, clear, resume, crawl start, mode switch or
configuration change alters what your queries return, or what your next
crawl does, without any signal in your own calls. Serialise state changes across sessions; only read-only work runs
in parallel (see "Sharing one instance between sessions" in SKILL.md).

When a crawl is occupying the instance and you need it for something else,
the only two levers are **letting the crawl finish** and **clearing it**.
Pausing did not release it on 24.x (see the `SpiderPausedState` entry above), so
pause-load-resume is not a workaround to re-attempt. Plan around the
contention rather than trying to slip past it.

---

## Side effects on the user's machine — and on other sessions

This server drives a desktop application on someone's computer. Several tools
change what they see or lose their work:

- `sf_load_crawl` replaces whatever crawl they had open, and upgrades a
  crawl stored by an older release in place.
- `sf_crawl` and `sf_crawl_from_list` start a real crawl — it evicts the
  loaded crawl, runs under the instance's current configuration when no
  `config_path` is given, is stored whether you wanted it kept or not, and
  blocks most tools until it ends. Read `crawlTotalLimit` before starting
  one; never fire it speculatively against a large site.
- `sf_set_config_spider` changes settings that outlive the session; restore
  what you changed.
- `sf_set_crawl_mode` needs the loaded crawl cleared, and swaps the eight
  per-mode settings.
- `sf_resume_crawl` starts fetching a paused crawl's queue.
- `sf_clear_crawl` empties the instance. Safe for a database-stored crawl,
  which survives in `sf_list_crawls`; destructive for one with no durable
  home. Check which you have, then check again afterwards — and copy any
  working files you still need out of the base directory first, because
  the script runner that reaches them is gated on a loaded crawl.
- A call to a tool name the server does not have drops the loaded crawl (no
  error names the cause; the next call fails with `You need to load or run
  a crawl first`). **[24]** So did a heavy export that timed out.
- `sf_connect_api` / `sf_disconnect_api` change which API data the next
  crawl collects.
- `sf_write_text_file` overwrites without warning.
- `sf_open_url_in_browser` opens a tab on their desktop.

Every item on that list lands on every other session sharing the instance
as well, silently — the other session sees changed data or a load-first
error, never a message. Give each session its own subfolder under the base
directory and announce state changes before making them (see "Sharing one
instance between sessions" in SKILL.md).

None of this is destructive to source data, but on an unattended machine it
is disruptive and can silently discard hours of crawling. Prefer read-only
tools, keep working files in a dedicated subdirectory, and check before
starting, resuming or clearing a crawl.

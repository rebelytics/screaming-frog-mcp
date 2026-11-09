# Gotchas and boundaries

Reference file for the `screaming-frog-mcp` skill (CC BY 4.0 — Eoghan Henn and contributors). Things that error, mislead, or cannot be done at all. Ordered roughly by how
early in a session they bite.

Load triggers: before the first export; a call errors unexpectedly, or a running crawl refuses queries; an empty filter or row export disagrees with a summary count

## Contents

- Errors you will hit
- Silently misleading values
- Hard boundaries — no workaround
- Side effects on the user's machine — and on other sessions

---

## Errors you will hit

**`IllegalStateException: UNEXPECTED_MESSAGE loadCrawl in state: SpiderPausedState`**
The desktop app is holding a paused crawl and will not load another over it.
Do **not** resume it to make the load succeed — that discards someone's
in-progress work. Call `sf_crawl_progress`: it reports the `databaseId` in
memory, and very often that is already the crawl you wanted, in which case
query it in place and skip the load entirely.

**Pausing a running crawl does not help either** — the same exception is what
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

**`Crawl is open in another SEO Spider instance`**
The crawl you asked `sf_load_crawl` for is open in the desktop UI. A crawl
run or opened in the UI stays locked to that instance, and the MCP cannot
load it until the user closes it there. This recurs on every UI→MCP
handover, so state it up front — "close the crawl in the app, then I'll
load it" — rather than rediscovering it per attempt.

**`IOException: You need to load or run a crawl first`**
No crawl is loaded. Run `sf_load_crawl`. The non-obvious case: this appears
*mid-session* after the user restarts the host application, on tools that
worked minutes earlier. Loaded state is not durable. Re-load rather than
diagnosing a broken server.

Two more paths into it. **The precondition is server-wide, not per-tool:
`sf_run_node_js_script` raises it too**, even for a script that only reads and
aggregates files on disk and touches no crawl data at all. Loading any crawl
satisfies the gate — a one-URL throwaway is enough — so keep one loadable when
doing pure file work. And **an export timeout drops the loaded crawl**, so
this error frequently arrives as the *second* symptom of a failure whose first
symptom was a timeout: re-load before concluding anything about the next call.

**A `sf_generate_bulk_export(category: "Queued URLs")` that never returns**
The Queued URLs export times out at the MCP layer on a large queue (observed
twice on a ~1M-URL list-crawl queue). The work aborts with the request: no
partial file, nothing in the destination directory or the app's temp
directory, and the loaded crawl is gone as well — the next call fails with
`You need to load or run a crawl first`. There is nothing to salvage and
retrying reproduces it. Route large queue exports through the desktop UI
instead. The size is what matters, not the category: on a queue of a
hundred-odd rows the same export returns in seconds, on a crawl where
`Links: All Inlinks` timed out — so read `waiting` from `sf_crawl_progress`
first and let it decide the route.

**Any heavy export that times out drops the loaded crawl — for every
session on the instance.** The Queued URLs case above is one instance of a
general behaviour: a `Links: All Inlinks` bulk export on a ~100k-URL crawl
timed out the same way, after which `sf_crawl_progress` reported
`SpiderNoDataIdleState` under a different `databaseId`. Any other session
driving the same instance then fails with `You need to load or run a crawl
first` on its next call, with no way to tell that a timeout elsewhere
caused it. Re-load, tell the other session, and route the export through
the desktop UI (see "Sharing one instance between sessions" in SKILL.md).

**`Can't run tool … as SEO Spider is busy`**
A crawl is active in the instance (`stateName: SpiderActiveState`, typically
one you started with `sf_crawl`), and every query tool — `sf_url_info`, the
exports, the reports — is refused while it runs. `sf_crawl_progress` still
answers. The loop is `sf_pause_crawl` → query → `sf_resume_crawl`, which
behaved cleanly on every repetition. This is not the `SpiderPausedState`
refusal above: that one blocks *loading* over a paused crawl, this one
blocks *reading* from an active crawl, and pausing fixes only the second.

**`IOException: The following name/s are invalid ["…"]. The valid names you can use are: […]`**
Data-field names are **per element *and* filter**, not global. `Unique
Inlinks` is valid on `Internal`/`HTML` and invalid on `Response Codes`/
`Internal Blocked by Robots.txt`. The error enumerates the valid set, so this
costs one round trip rather than a wrong answer — but
`sf_list_available_data_fields_for_seo_element_and_filter` avoids it.

**`IOException: Not configured to store page content`**
`sf_bulk_export_page_content` on a crawl run without Store HTML. The setting
was chosen at crawl time and nothing in `sf_list_crawls` reveals it
beforehand, so this is discovered by trying. It is not a reason to ask for a
re-crawl: check what the crawl *does* hold first. `Links: All Outlinks` with
the `Link Path` and `Link Position` fields recovers per-anchor destination,
anchor text and the emitting XPath — the element-level attribution the
content export was usually wanted for — on the same crawl that refused the
export.

**`NoSuchFileException: /…/wk/out.ndjson`**
`file_path` does **not** create parent directories. Call
`sf_create_directory` first. The path in the error is the absolute path on
the user's machine, which is also a quick way to confirm the base directory.

**`Status Code: 0` on the start URL, `Indexability Status: No Response`, and
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

**`sf_crawl_progress` is scoped to the MCP's own instance and does not name
the crawl it describes.** While a crawl the user started in the desktop UI
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
acted on none of it. The known cause of the second: the `config_path`
profile was saved while the desktop app was in List mode. The Spider > Crawl
tab holds separate **Crawl** and **Store** checkboxes per link type, and a
Store-without-Crawl state produces exactly this signature; `sf_crawl` is
spider-only, so a List-mode profile silently yields a one-URL spider crawl.
The outlink count on the start URL looks healthy in both the working and the
broken case (123 recorded outlinks in the broken one), so it cannot settle
the question. After re-saving the profile in Spider mode, the same call
climbed past 6,800 URLs. See "config_path provenance" in `tool-surface.md`.

**Exports return NDJSON inline when `file_path` is omitted.** No error, no
truncation warning — just a very large tool result. This is the default
behaviour and the single most expensive mistake available in this server.
See "Getting data out without flooding the context" in SKILL.md.

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

**An empty filter raises an exception — and "empty" has two very different
causes.** `IllegalArgumentException: start_index is greater than the total
number of URLs of 0` means the filter holds zero rows. That is **not** the
same as "zero exist". Either:

- there genuinely are none; or
- **the crawl's configuration never collected or never stored them.**
  Screaming Frog's "Show external URLs blocked by robots.txt" is off by
  default: the crawl *counts* those URLs but does not keep them, so the filter
  is empty while the true figure is not. Equally, if resource crawling or JS
  rendering was off, the CSS/JS/image filters are empty by construction.

**Sibling tools signal "empty" through different channels — bulk exports
return a success-shaped string, not an exception.** `sf_generate_bulk_export`
signals an empty report by returning the plain string `"Report generated no
data"` — no error raised, no file written — where
`sf_export_seo_element_urls` throws the `IllegalArgumentException` above for
the same condition (both observed on one live 62.5k crawl). Never treat the
absence of the known error signature as data present: check the result
string before assuming a file exists. The same two-cause caveat applies —
"no data" means a true zero OR an unpopulated filter; cross-check against
Crawl Overview exactly as below. Treat both empty signatures identically:
as *unproven*, never as zero.

Two harder lessons from a 55k-URL verification session:

- **When a bulk export is empty, re-derive the same population through a
  per-filter inlink export before concluding anything.** The two paths are
  not equivalent: the non-indexable-page-inlinks bulk export returned
  `Report generated no data` while the per-filter inlink exports for the
  same underlying rows returned 140 and 1,451 records.
- **The Crawl Overview summary counts independently of the display layer,
  so a non-zero summary beside an empty row export is not a contradiction
  to debug.** The server reported 787 external-blocked-by-robots URLs in
  the summary while both the bulk export and the row-level filtered export
  for the same population came back empty (the latter as the
  `IllegalArgumentException` above), because a crawl-configuration setting
  governs whether that URL class is *displayed* at all — the summary counts
  it regardless. A disagreement between the two surfaces locates a
  configuration boundary, not a bug, and **the summary is the count to
  trust**.

**Bulk-export receipts: read the "Export saved to …" line, not the `path`
field.** The receipt's `path` field points to an internal temp file
(`.ScreamingFrogSEOSpider/temp/tmpCrawlOverviewReport`); the actual saved
location appears only in the trailing "Export saved to …" text. Scripts
that read the `path` field process the wrong file.

**Prove the filter could have been populated before you report zero.**
`sf_generate_report(category: "Crawl Overview")` carries totals the filters
don't. In one crawl it reported `Total External blocked by robots.txt: 1,158`
next to `URLs Displayed: 61,393` — exactly 1,158 short of the 62,551
encountered — while the matching filter raised the empty-set exception. Also
check the parent population: a crawl holding 0 CSS and 0 JS URLs cannot report
a blocked script whatever the truth.

Reporting an unpopulated filter as a clean zero is the worst outcome available
here, because it reads as a verified all-clear rather than as missing data.

**The `Rel` column in inlink/outlink exports is empty for arbitrary link
relations — including `rel="alternate"`.** On a multi-domain crawl, `Rel`
comes back `null` on every link row, country-switcher anchors included,
though they verifiably carry `rel="alternate"` in the page source. The
column appears to be populated only for the link relations the crawler
treats as directives (nofollow and similar), not for arbitrary `rel`
values — so a filter on it to select switcher or alternate links returns
nothing, and in a link-counting task that empty result reads as "the
component emits no links". Never build a filter on `Rel` for arbitrary
relations; classify the emitter with **`Link Position`** (Navigation for
switcher anchors vs Head for hreflang link elements) and pin the component
with **`Link Path`**, cross-checking the two. The general rule: a column
existing in an export schema is not evidence it is populated for the
values you care about — before filtering on any field, check its
distribution across a known-positive population, because a field that is
null where you know the attribute exists will fail silently as absence
rather than loudly as error.

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

**No `sf_*` tool runs a list-mode crawl — the application's own CLI does,
and the script runner reaches it.** `sf_crawl` takes a single `crawl_url`
and starts a spider crawl. No parameter accepts a URL list, and the tool
surface contains no list-mode equivalent — verified by enumerating every
tool, not inferred from one failure. Saved crawls do report
`spiderMode: "List"`, so the desktop application supports it; the MCP
simply doesn't expose it as a tool.

This is a genuine gap in the official server and worth reporting upstream,
because list mode is the natural fit for verification work — checking a
specific set of URLs after a fix.

*Consequence for planning:* if the user asks for a list crawl, don't
improvise one by starting a spider crawl and stopping it, and don't declare
list mode unreachable. `sf_run_node_js_script` can spawn the SEO Spider's
own command-line launcher on the user's machine with `--crawl-list <file>
--headless`, which runs a genuine List-mode crawl with exports and a saved
crawl (recipe in `workflows.md`, "Re-checking a list of URLs"). Two rules
travel with it. **A CLI launched from a script inherits factory defaults,
not the user's crawl setup** — no rendering, no auth, no custom extraction —
so look for a saved `.seospiderconfig` profile — saved in List mode — and
pass it with `--config` before reading a failure as a property of the
site. And **it is a separate
process**: the MCP's loaded-crawl state is untouched, and the runner still
needs some crawl loaded to launch at all. Where only a status re-check is
needed and the site does not challenge non-browser clients, the lighter
detached Node sweep in the same recipe still works; where the edge serves a
bot challenge, that sweep gets the same block as any other non-browser
client and is no substitute for the CLI with a rendering profile.
Otherwise, either have the user start the crawl in the desktop UI and then
take over via `sf_load_crawl`, or — for page-level questions like meta
robots, canonicals, hreflang or which links a template emits — fetch the
pages another way. A browser or HTTP client reads raw source perfectly
well. **But note it will ignore robots.txt**, so it can confirm what a page
contains and cannot confirm whether a URL is blocked. Blocking is a
robots.txt evaluation, and this server already answers it via
`Matched Robots.txt Line`.

**Where your shell cannot see the base directory, files move only through
the server.** It reads and writes one fixed root on the SEO Spider's machine.
From an agent on another machine, use `sf_write_text_file` /
`sf_read_text_file` / `sf_run_node_js_script`: you cannot hand it a local
script by path or open its exports directly. On the same machine your shell
reads the root directly, but a script that prints only what you need is cheaper.

The confinement applies to the *file tools*, not to code the server
executes: a script run with `sf_run_node_js_script` can write outside the
allowed base directory. That is the cheapest route for moving bulk output
into a shared location the session can read directly — a zero-context file
handoff. Similarly, `sf_crawl`'s `config_path` accepts an absolute path
outside the base directory.

**`sf_run_node_js_script` runs a file, not a string.** Write it first with
`sf_write_text_file`. `args` must not begin with `-`.

**Only one config file can reach `sf_crawl`.** `config_path` takes the
ordinary crawl config profile; the separate `.seospiderauthconfig` that the
CLI accepts as a second parameter cannot be supplied at all — the MCP
exposes one config parameter where the CLI has two. Anything that must reach
an unattended crawl has to live in the ordinary profile, which is why
HTTP-header auth survives an MCP-started crawl and app-saved credentials do
not (see "Authentication for MCP-started crawls" in `tool-surface.md`).
The limit is `sf_crawl`'s, not the application's: a CLI crawl spawned from
the script runner takes the CLI's own option set, which the vendor documents
as including a separate auth-config option — unexercised, so treat it as a
route to try, not a verified one.

**One crawl at a time — per instance, not per session.** Every query runs
against whatever is loaded. There is no per-call crawl scoping, so
comparing two crawls means loading one, exporting, loading the other,
exporting, and diffing outside the server. And the loaded state is shared
by every MCP client driving the instance: a second session cannot hold a
different crawl, and its load, clear, resume, `sf_crawl` or timed-out
export changes what your queries return without any signal in your own
calls. Serialise state changes across sessions; only read-only work runs
in parallel (see "Sharing one instance between sessions" in SKILL.md).

When a crawl is occupying the instance and you need it for something else,
the only two levers are **letting the crawl finish** and **clearing it**.
Pausing does not release it (see the `SpiderPausedState` entry above), so
pause-load-resume is not a workaround to re-attempt. Plan around the
contention rather than trying to slip past it.

---

## Side effects on the user's machine — and on other sessions

This server drives a desktop application on someone's computer. Several tools
change what they see or lose their work:

- `sf_load_crawl` replaces whatever crawl they had open.
- `sf_crawl` starts a real crawl — it evicts the loaded crawl, runs under
  the instance's current config when no `config_path` is given, has no URL
  cap you can assume, and refuses every query tool until paused. Never fire
  it speculatively against a large site.
- `sf_resume_crawl` starts fetching a paused crawl's queue.
- `sf_clear_crawl` empties the instance. Safe for a database-stored crawl,
  which survives in `sf_list_crawls`; destructive for one with no durable
  home. Check which you have, then check again afterwards — and copy any
  working files you still need out of the base directory first, because
  the script runner that reaches them is gated on a loaded crawl.
- A heavy export that times out drops the loaded crawl (no error names the
  cause; the next call fails with `You need to load or run a crawl first`).
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

# Workflows

Reference file for the `screaming-frog-mcp` skill (CC BY 4.0 — Eoghan Henn and contributors). How to get answers out of a large crawl without drowning in rows.

Load triggers: any multi-step analysis (the export→status→aggregate→summarise loop); an absence-based conclusion (queued URLs and the other subtraction traps); attributing links or blocked URLs to a template component or robots rule; crawling or re-checking a list of URLs, pulling their page text to disk, or confirming the newest band live; declaring a capability missing before checking the application's CLI; canonical verification or a stored crawl's API date range

## Contents

- The core loop
- Counting things
- Attributing findings to the template that causes them
- Attributing blocked URLs to the robots.txt rule
- Reconciling to a total
- Reproducing a pattern someone saw in the desktop UI
- When an export is refused, ask what the crawl does hold
- Re-checking a list of URLs
  - The default — a list crawl in the MCP's own instance
  - Route A — a List-mode crawl through the SEO Spider's CLI
  - Route B — a detached status sweep
- Absence and subtraction traps
- Verifying a canonical deployment across a template
- Reading a stored crawl's API date range and filters
- Confirming a single claim cheaply

---

## The core loop

Five steps. Deviating from them is what makes crawl analysis expensive.

**1. Prepare a working directory.** Writes don't create parents.

```
sf_allowed_base_directory()             → the one writable root
sf_create_directory(path: "wk")
```

**2. Export to a file.** The export is a background task; `file_path` is
required.

```
sf_export_seo_element_urls(
  seo_element_name: "Response Codes",
  filter_name: "Internal Blocked by Robots.txt",
  data_fields: ["Address", "Inlinks", "Matched Robots.txt Line"],
  file_path: "wk/blocked.ndjson")
```

The call returns only `The task '…' has started and is currently running`.

**3. Fetch the result with `sf_task_status`.** While the task runs it shows
progress and every other tool is refused; once it reads `SUCCEEDED` the
result is a receipt, not the data:

```json
{"format":"NDJSON","path":"…/wk/blocked.ndjson","total_records":2409,"sample":"{…}"}
```

Confirm the `path` is the file you asked for before reading the count — the
status describes the last task that started, and a call that failed outright
(an empty filter does) leaves the previous task's receipt showing. For
reports and bulk exports the saved location is the `Export saved to …` line
instead (`gotchas.md` has the shape per tool). One export in flight at a
time; a task that outgrows its purpose is stopped with `sf_cancel_task`.

Name only the `data_fields` you need. Omitting the parameter exports every
field, which bloats the file and slows the aggregation for no benefit.

**4. Write an aggregation script and run it.** The script must exist on disk
before it can run, so write it first. Node built-ins are enough.

```js
const fs = require('fs');
const rows = fs.readFileSync(__dirname + '/blocked.ndjson', 'utf8')
  .split('\n').filter(Boolean).map(JSON.parse);

const by = new Map();
for (const r of rows) {
  const k = r['Matched Robots.txt Line'] || '(none)';
  const c = by.get(k) || { urls: 0, inlinks: 0 };
  c.urls++; c.inlinks += r.Inlinks || 0;
  by.set(k, c);
}
for (const [k, v] of [...by].sort((a, b) => b[1].inlinks - a[1].inlinks))
  console.log(`${String(v.urls).padStart(6)} urls  ${String(v.inlinks).padStart(8)} inlinks  ${k}`);
```

```
sf_write_text_file(path: "wk/agg.js", content: <the script>)
sf_run_node_script(script_path: "wk/agg.js")
```

**5. Read only the aggregate.** Stdout is the tool result, so have the script
print the finished table — not the rows behind it.

Why this matters: a single blocked-links report was 134,894 rows. Read
back row by row, that is an unusable amount of context. Through this loop the entire
decomposition cost a few dozen lines of output.

---

## Counting things

There is no count tool, but `total_records` in the export receipt is one.
Export to a file with a single `data_fields` entry, fetch the receipt with
`sf_task_status` and read the number. This
is the cheapest question you can ask the server.

**Do not reach for `max_rows` to probe a count.** It caps `total_records` in
the receipt, so `max_rows: 1` reports `total_records: 1` against a 2,409-row
set — a number that reflects your parameter rather than the data, and looks
exactly like a real answer. Run the export uncapped.

**"Are there any at all?" is a two-step question.** A filter matching nothing
fails the call with `No URLs match filter '…' …, so there is nothing to
export` (an empty bulk export instead succeeds with `Report generated no
data`). Either means *this selection holds no rows* — which is only
the same as "none exist" if the crawl was configured to collect them. Confirm
with `sf_generate_report(category: "Crawl Overview")`, which reports totals the
filters don't store, and by checking that the parent population is non-empty.
Then answer "there are none" or "the crawl can't tell us" — they are different
answers and only one of them is an all-clear.

**Parse the Crawl Overview report as one whole document.** It is not NDJSON —
`sf_generate_report` writes a single pretty-printed JSON object regardless of
the extension you asked for, so the per-line idiom from step 4 above throws a
SyntaxError on the first line. Use one `JSON.parse` of the whole file. Its
`summary` block reconciles the category totals for you (internal
non-indexable = blocked + noindex + redirect targets), which is often the
entire cross-check in one read.

The shape inside, so no throwaway script is spent rediscovering it (observed
on one stored crawl under 24.x — two sessions burnt three scripts
between them on it; on 25 the document's six top-level objects are
`crawlMetadata`, `summary`, `seoElements`, `depthStats`, `topLinks` and
`responseTimes`): the per-filter counts live at `seoElements.seoElementFilterCounts`,
an array of `{seoElementName, filterCounts: [{filterName, urlCount,
totalUrls, percentageOfTotal}]}`; the `summary` block's keys are named
objects of the form `{description, urls, …}` rather than bare numbers, so
read `.urls` off each one. Sniff the first level once with a small
`sf_read_text_file` if a different version disagrees.

---

## Attributing findings to the template that causes them

This is the highest-value thing the inlink exports support, and it is easy to
miss because it depends on two columns nobody thinks to request.

Inlink exports carry **`Link Path`** (the XPath of the emitting element) and
**`Link Position`** (Navigation / Content / Header / Footer). Together they
turn "there are 45,000 bad links" — which no developer can act on — into
"one component on the product template emits 99.8% of them", which is a
ticket.

**Method.** Strip the positional indices from the XPath to get a template
signature, then group:

```js
const sig = x => x.replace(/\[\d+\]/g, '[]');
```

Then report, per bucket: link count, **count of distinct destination URLs**,
count of unique source pages, the links-per-source-page ratio, and the top
signatures with their share.

**The links-per-page ratio classifies the bucket before you open any code:**

| Ratio | Means |
|---|---|
| ~1.00 | a single site-wide chrome element |
| ~2.00 | one chrome element rendered by two partials (typically desktop + mobile) |
| non-integer > 1 | a per-item content component |

**Distinct-destination count is the other discriminator.** A large link count
resolving to a *small closed set* of targets changes the question entirely —
from "how do we stop linking to these" to "why are these not legitimate
destinations". Tens of targets means an enumerable taxonomy; thousands means
a generated surface.

**Check source and destination separately.** It is easy to describe links by
the page type they point *at* when the actionable fact is the page type they
are emitted *from*. Group by both; they are frequently different, and only
the source side tells anyone which template to edit.

**`Link Position` also separates two emitters of the same destination — and
do not reach for `Rel` instead.** One destination reached from both a
country-switcher component and the head's hreflang annotations splits
cleanly on `Link Position` alone (`Navigation` vs `Head`), with `Link Path`
pinning the exact component in each — and the split is itself the finding.
The `Rel` column looks like the natural classifier for `rel="alternate"`
anchors and is null for them in practice (see `gotchas.md`), so a Rel-based
filter silently returns nothing.

---

## Attributing blocked URLs to the robots.txt rule

Request `Matched Robots.txt Line` on a blocked-by-robots filter and group by
it. The output is a table of which rule is costing what:

```
   120 urls    90000 inlinks  Line:4 /*/results
  2200 urls    45000 inlinks  Line:9 /*?*viewRef=
```

Read that table for **rules catching more than they were written for**. A
rule that produces far more URLs than its author would expect, or that
appears against URLs of an unexpected type (product pages under a rule meant
for search endpoints), is an over-broad pattern. Robots patterns are prefix
matches with no implicit end anchor, so an unanchored path rule matches any
URL where the token appears after a path segment — including content slugs
that merely begin with the same word.

When you suspect that, test the proposed replacement rule against the actual
URL set before recommending it. A small matcher implementing `*` and `$`
semantics, run over the exported addresses, gives you the confusion matrix:
still-blocked, newly-released, newly-caught. That converts a recommendation
into a measurement.

---

## Reconciling to a total

When decomposing a report into buckets, make the parts sum to the whole and
say so. If the buckets total 133,751 of 134,894, the missing 1,143 is a real
group you haven't characterised — and it is exactly where surprises live. In
one case that residual contained legitimate product pages caught by an
over-broad rule, which turned out to be the most valuable finding in the
analysis.

A decomposition that doesn't reconcile is a sample presented as a census.

The mirror-image check applies to any rate or "out of N" that leaves the
session: resolve what population N actually covers before quoting it. The
crawl's headline total unions internal and external URLs, so "out of 55k
crawled URLs" can describe a site of 15k pages — see the denominator entry
in `gotchas.md`.

---

## Reproducing a pattern someone saw in the desktop UI

When a human reports a pattern from the SEO Spider's interface — "about a
hundred redirects, and they all land on four noindexed pages" — reproduce it
from the export before building anything on it. A UI view is a query whose
parameters are not part of the reported result: an active filter, a search
box with text still in it, or a sort on a truncated column is invisible in
the verbal report, and the reporter is not withholding it, they cannot see it
either. In one case a confident, specific claim of that shape was
contradicted by the export (112 redirects, 112 distinct targets, zero
noindexed) because a filter was live in the user's view.

The check costs one export and one group-by. If it fails to reproduce, ask
which tab, filter and search were active rather than debating the numbers —
that is the fastest route to the real finding, which is usually adjacent to
the reported one.

**Say so when the claimed provenance is structurally impossible.** In a
list-mode crawl, redirect *targets* are off-list and never fetched, so the
crawl holds no indexability data about them at all — a claim about their
noindex status cannot have come from that crawl whatever the numbers say.
Flag that as impossible provenance rather than merely doubting the figures:
it identifies the wrong source immediately, where a numeric disagreement
invites a second opinion on the same bad data.

## When an export is refused, ask what the crawl does hold

A refused export is a statement about how the crawl was configured, not about
what it knows. Before asking anyone to re-crawl — expensive, and often
impossible on a large site — check whether another report was populated
anyway and carries the same fact by a different route. The worked example:
`sf_bulk_export_page_content` fails outright when the crawl stored no HTML,
but `Links: All Outlinks` with `Link Path` and `Link Position` was populated
on that same crawl and gave per-anchor destination, anchor text and the
emitting XPath — component-level attribution without a byte of stored
content. Re-crawling is the last option, not the first.

The same question — "what does this crawl hold that nobody asked it for?" —
pays off on multi-domain estates. With hreflang set to crawl, a spider crawl
of one domain fetches the alternates it declares on every sibling domain,
and they land under `External > HTML` **with status codes** — on a large
site, thousands of fetched alternates per sibling domain from one crawl of
a single reference domain. That is a status-verified
cross-domain existence map — which alternates resolve, which 404, which
redirect — from a single crawl, not merely the annotation list the
`Hreflang` element reports. Check `External > HTML` before starting a crawl
per domain.

## Re-checking a list of URLs

"Give me the current status of these 1,700 URLs" is a list crawl, and since
25 the server runs one itself. Fetching the URLs from a remote agent's
sandbox is usually worse: a remote agent's egress collects 429s and WAF
challenges within a few hundred requests, where the user's machine has an
ordinary network identity the edge is not filtering. Three routes, in
order of preference: the list-crawl tools in the MCP's own instance; route
A, the application's CLI spawned from `sf_run_node_script`, for a list
crawl that must leave the instance's loaded crawl and configuration alone;
and route B, a lighter status-only sweep for sites that do not challenge
non-browser clients.

### The default — a list crawl in the MCP's own instance

```
sf_crawl_progress()                      → note what is loaded; it will be replaced
sf_clear_crawl()                         → only if a crawl is loaded and the mode must change
sf_set_crawl_mode(mode: "LIST")          → reply lists the per-mode settings now in effect
sf_get_config_spider(category: "crawlConfig")   → read before changing anything
sf_set_config_spider(config: "{…}")      → rendering, Store HTML, extraction: only what the job needs
sf_crawl_from_list(list_mode_file: "<absolute path>/wk/list.txt", crawl_name: "…")
sf_crawl_progress()                      → poll until crawlRunning is false
```

- **The list:** inline in `list_mode_urls` (one URL per line) for a short
  one; for a long one write the file with `sf_write_text_file`, or have a
  script write it from an earlier export so no URL bytes cross the context,
  and pass its **absolute** path — `list_mode_file` is not resolved against
  the base directory (`sf_allowed_base_directory` gives the prefix).
- **Configuration after the mode switch, not before:** eight settings are
  kept per mode and swap on the switch (`tool-surface.md`, "Crawl mode and
  list crawls"). In LIST mode the observed defaults limit crawl depth to
  the listed URLs.
- **The crawl replaces the loaded crawl and is stored.** Name it so the
  user can find it, and list it in the session-end handover.
- **It uses the instance's network identity and configuration**, so a site
  that challenges non-browser clients needs `crawlerMode: "RENDER"` set
  first. Rendering is slow: about seven seconds per URL at default speed on
  one site.
- **Page text:** switch `storeOriginalHtml` / `storeRenderedHtml` on before
  the crawl, then `sf_bulk_export_page_content` with `MARKDOWN` or
  `VISIBLE_TEXT` writes one row per page to disk. Its status receipt
  carries a whole page as the sample.
- **Restore the configuration afterwards** and read it back.

### Route A — a List-mode crawl through the SEO Spider's CLI

Everything in this route was observed on 24.x and has not been re-run on
25. Use it when the list crawl must run as a separate process.

The desktop application ships a command-line launcher (on macOS,
`/Applications/Screaming Frog SEO Spider.app/Contents/MacOS/ScreamingFrogSEOSpiderLauncher`;
the vendor's CLI documentation names the Windows and Linux equivalents).
Spawned detached from a launcher script run by `sf_run_node_script`, it
runs a genuine List-mode crawl:

```
spawn(<launcher>, [
  '--crawl-list', <list.txt>, '--headless',
  '--output-folder', <dir>, '--overwrite',
  '--export-format', 'csv',
  '--export-tabs', 'Response Codes:All',
  '--save-report', 'Redirects:All Redirects',
  '--config', <profile>.seospiderconfig,      // omit = factory defaults; List-mode profiles only
  '--save-crawl'                              // optional: also writes crawl.seospider to the output folder
], { detached: true, stdio: ['ignore', logFd, logFd] }).unref()
```

Observed: a few hundred URLs crawl in seconds, about a minute of wall clock
including JVM start; a ~2k-URL list takes on the order of a quarter of an
hour; the exports named in `--export-tabs` / `--save-report` land in the
output folder and the process exits cleanly. Poll the log file with
`sf_read_text_file`. Point `--output-folder` at a location the session can
read directly (the spawned process is not confined to the base directory)
and no copy step is needed.

Five properties decide whether the result means what it seems to:

- **It is a separate process.** It ran beside the MCP's own instance with
  no instance conflict, and the MCP's loaded crawl is untouched. The runner
  itself still needs some crawl loaded to launch anything, and a crawl the
  CLI saved is read back the ordinary way — `sf_load_crawl`, then the
  export tools.
- **No `--config` means factory defaults, not the user's setup** (the log
  says so: `No List mode default config file found`). Factory defaults give
  titles and descriptions with pixel widths, H1, canonical, indexability,
  response codes and hreflang, but no JavaScript rendering, no stored auth
  and no custom extraction — a value that lives in a `dataLayer` or a
  rendered component is simply absent. On one site behind a bot challenge
  the default-config crawl got a 403 on nearly every URL, while the user's
  saved profile with rendering enabled answered every one. Before reading
  a wall of 403s or empty fields as a property of the site, look for the
  project's saved `.seospiderconfig` (often beside the MCP's base
  directory, or ask) and re-run with `--config`.
- **A profile saved in Spider mode turns the list run into a spider crawl
  seeded by the list** — the mirror of the List-mode profile trap under
  `sf_crawl` (`tool-surface.md`, "config_path provenance"), and just as
  silent. Observed on one run: with a Spider-mode profile the log's queue
  (`mWaiting`) held about 61k URLs after the first ~1,000 of a ~10.5k list,
  because the listed pages' outlinks and resources were queued; the same
  list without `--config` saw the queue fall with the list and the run
  exit cleanly. The first progress poll is a scope check: compare `mWaiting`
  with the list size, and kill the process when the queue exceeds what is
  left of the list. Pass only a profile saved with the app in List mode.
- **Every CLI run lands in the user's project list as a stored crawl,
  with or without `--save-crawl`** — observed on one version at last
  verification: two list runs without the flag, one of them killed, both
  appeared in `sf_list_crawls` named after the list file. With the flag
  and `--output-folder` the run also writes a full `crawl.seospider` file
  into the output folder (observed: about 400 MB for under 2k URLs on a
  rendering profile). The project-list copy is the one `sf_load_crawl`
  reads, so keep the output folder out of synced or shared project
  folders, or move that file out when the run ends. No tool deletes a
  stored crawl (`tool-surface.md`, "Deleting stored crawls"), so name
  every CLI run, aborted ones included, in the session-end handover for
  deletion in the Crawls dialog, and tell the user which run is which.
- **It is not the detached HTTPS sweep.** Route B below fetches with a
  plain Node client, which an edge that challenges non-browser clients
  blocks exactly as it blocks a remote sandbox; the CLI with a rendering
  profile is the route past such a challenge.

**The same route carries page text, not only status.** When a job needs
the content of pages a site challenges, run the list crawl on a profile
that stores rendered HTML, then load the saved crawl and write the text to
disk with `sf_bulk_export_page_content` and `file_path`, so it never
crosses the context. Store HTML is a crawl-time setting the crawl listing
does not reveal, so confirm the profile has it before launch (in the app,
or ask the user), and save a variant profile rather than editing theirs.
This pairing was not exercised end to end at last verification: check the
first export's size and a sample page before building on it.

**Check free disk before launch, and poll for the guard while it runs.**
The SEO Spider stops itself when free disk falls below an absolute
threshold (5.0 GB on the instance observed): the log shows `Disk usage:
… Less than 5.0 GB left`, then `FATAL - SeoSpider failed to start … Disk
usage too high`, and nothing reaches the session unless the poll looks
for it.

- In the launcher, read free space with `fs.statfsSync` and refuse to
  start below about 5 GB plus, for a rendering profile that saves
  screenshots, about 0.5 MB per URL — an estimate from one observed run,
  where per-page screenshots grew the stored crawl by 0.4–0.5 MB per HTML
  page, not a vendor figure. Bulk exports of the same crawl (raw HTML
  runs to hundreds of MB) draw on the same disk.
- Never run two rendering crawls in parallel on one disk; it doubles the
  growth rate.
- Poll the log for `FATAL` and `Disk usage` as well as for progress; a
  poll that greps only progress lines shows a run that stopped moving and
  never why.
- Recovery: a crawl the guard aborted stays in `sf_list_crawls` as
  paused at the percentage it reached, loads normally with
  `sf_load_crawl`, and exports every URL it reached. Build the remainder
  from the list minus the exported addresses and run only that.

### Route B — a detached status sweep

For a status-only re-check on a site that does not challenge non-browser
clients, a throttled Node worker is lighter than a crawl.

**1. Export the input from the crawl that already holds it.** A stored list
crawl is a URL inventory even when its status data is out of date.

```
sf_load_crawl(...)
sf_export_seo_element_urls(
  seo_element_name: "Internal", filter_name: "All",
  data_fields: ["Address"], file_path: "wk/urls.ndjson")
```

No URL bytes pass through context — only the receipt's `total_records`,
read from `sf_task_status`.

**2. Write a launcher that spawns a detached worker.** The launcher starts
the worker with `spawn(..., { detached: true, stdio: ['ignore', logFd,
logFd] })`, calls `unref()`, prints the child PID and exits. The worker
walks the list with throttled `https.get` checks (keep concurrency low —
this is someone's real connection and no robots.txt is being consulted),
appends one result per line to a results file, and rewrites a small progress
file after each batch.

```
sf_write_text_file(path: "wk/sweep.js", content: <worker>)
sf_write_text_file(path: "wk/launch.js", content: <launcher>)
sf_run_node_script(script_path: "wk/launch.js")   → returns in ~1s
```

The tool call's time budget bounds the *launcher*, not the sweep, so a run
of any length is possible.

**3. Poll, then aggregate.** `sf_read_text_file` on the progress file until
it reports done, then a second script over the results file that prints only
the status-code breakdown and the rows that changed.

Two preconditions: the runner is gated on a loaded crawl even for work that
touches no crawl data, so keep any crawl loadable; and the worker's own
writes are not confined to the allowed base directory, so it can drop the
aggregate somewhere the session reads directly.

One boundary on what the sweep is for: it re-checks the *status* of URLs
the crawl already holds. Do not extend it into a live re-fetch whose counts
get subtracted from the crawl's stored counts — that joins two points in
time, and the next section says why it fails.

## Absence and subtraction traps

Three ways a crawl analysis produces a clean, plausible, wrong list. All
three came from one ~100k-URL crawl in one day; each was caught only by
opening the individual cases. They share a shape — a conclusion drawn from
what is *not* in the data, or from a difference between two counts — and
they are grouped here because the checks against them are cheap and the
same every time.

**1. Queued URLs are not absent URLs.** A crawl's URL exports contain
fetched URLs; its queue contains discovered-but-unfetched ones. Any
conclusion from absence — orphans, "not linked", "not reachable", a sitemap
diffed against `Internal: HTML` — treats a queued URL as missing when
discovery is exactly what the question asks about, and a queued URL was
*discovered via a link*. The queue is the tail of discovery, so it
concentrates the late-found, lightly-linked URLs an orphan analysis is
hunting for: in the case that produced this rule, every "orphan" the diff
found was a lightly-promoted item still in the queue, and the true answer
was zero. `sf_list_crawls` will not warn you — a hundred-odd waiting URLs
on a ~100k crawl rounds to `percentageComplete: 100.0`.

```
sf_crawl_progress()                      → read `waiting` (and stateName)
sf_generate_bulk_export(category: "Queued URLs", output_path: "wk/queued.ndjson")
```

If `waiting` is non-zero, export the queue and remove its rows from the
absent set before reporting anything. A queue of hundreds of rows exports
in seconds; on 24.x a queue of hundreds of thousands timed out and dropped
the crawl, and nothing that size has been tried on 25 (see `gotchas.md`) —
`waiting` tells you which you have, and `sf_cancel_task` stops an export
that runs too long. State in the deliverable whether the
crawl finished with an empty queue or the queue was subtracted.

**2. A live re-fetch joined against a crawl snapshot manufactures findings
wherever a listing rotates.** When the full inlink graph is too large to
export through the MCP, the tempting substitute is a subtraction: the
crawl's stored `Unique Inlinks` (total sources) minus a count obtained by
live-fetching one class of pages (e.g. the paginated listings) and counting
which items they link. Where the two sides are equal, the item is linked
only from that class. On a catalogue with sale, new-in, bestseller or
featured listings this is wrong in both directions: items that rotated
*into* a listing since the crawl inflate the live count (false "only from
this class"), items that rotated *out* deflate it (true positives hidden).
The failure shape: an item flagged as linked only from that class has
genuine other sources, and the live fetch found it on more listing pages
than the crawl did because it rotated into those listings between the
crawl and the fetch. Two further lessons:

- A one-sided sanity inequality (`live count > crawl total` is impossible)
  catches only the gross tail. Items inflated to *exactly* the total pass
  it, and moderate inflation below the total is invisible — the control
  certifies the plausible wrong answers.
- A consistency check against **a different field of the same snapshot**
  is stronger than any cross-time control. Here `Crawl Depth` did it for
  free: an item at depth 2 provably has a path that does not run through a
  listing page at depth ≥ 2, so any method flagging it is broken.

Rule: never subtract a live-fetched count from a crawl-stored count for a
page set containing rotating listings. Take both sides from the same
snapshot, or exclude the rotating listings from both, and say which. Where
the full `Links: All Inlinks` export is too large for the MCP, route it
through the desktop UI's export (File > Export or Bulk Export in the app,
then read the file with a Node script) rather than approximating it with a
live join.

**3. `Inlinks` and `Unique Inlinks` include self-links.** A template that
links to itself — breadcrumb tail, share links, a language switcher whose
current entry is an anchor — gives every page it renders a phantom source.
The shift is uniform, so it never appears as an outlier; it moves the whole
distribution by one, and a threshold near zero reads exactly inverted: the
"exactly one non-X source, fragile but safe" bucket was, for most members,
"zero plus self". The failure shape: a finding that no item depends on a
page class turns, once self is subtracted, into a non-zero count of items
that do — verify the corrected count on a stratified sample checked
against actual sources before reporting it. Any "inlinks from class X vs
not-X" subtraction subtracts self first:

```
sf_url_links(url: <item>, links_direction: "inlinks",
             links_category: "Hyperlink", file_path: "wk/src.txt")
```

then drop rows whose source equals the item, and separately label the
near-universal structural sources (an HTML sitemap page, a footer "all
items" list, sitewide menus) before counting what is left as editorial
linking — reachability and editorial linking are different numbers. The
cheap verification that makes this safe: **pull the actual inlink sources
of the single weakest item the method produced and classify each by hand**
(self / class X / structural / other) before trusting the aggregate. A run
that comes back perfectly clean is the one to check first. And when an
early spot-check turns up something odd but apparently harmless — one item
listing itself among its sources — write down what it would imply if it
were systematic before deciding it is minor; here it was in view hours
before it inverted the finding.

## Verifying a canonical deployment across a template

The question "did the canonical change ship on every page of the template,
and to the right host?" reduces to two exports and one aggregation. On a
large site, bound the crawl deliberately at a count comparable with the
pre-deploy crawl, so before and after compare (see "Bounding an
MCP-started crawl" in `tool-surface.md`).

```
sf_export_seo_element_urls(seo_element_name: "Canonicals",
  filter_name: "Canonicalised",
  data_fields: ["Address", "Canonical Link Element 1"], file_path: "wk/canonicalised.ndjson")
sf_export_seo_element_urls(seo_element_name: "Canonicals",
  filter_name: "Self Referencing",
  data_fields: ["Address", "Canonical Link Element 1"], file_path: "wk/self.ndjson")
```

(Resolve the exact field name with
`sf_list_fields_for_seo_element_filter` — it is
filter-specific.) Then aggregate in Node in two passes:

1. **By `(canonical host, same-path?)`** — does the canonical point at the
   expected host, and does its path equal the page's own path? Four cells
   (expected host + same path, expected host + different path, own host,
   other host) cover the deploy's intent and its two failure modes.
2. **By URL pattern class within each cell** — strip ids and slugs from
   `Address` to a template signature (the same idea as the XPath signature
   in the attribution recipe) and count per class. This is the pass that
   turns "a few dozen exceptions" into "three classes, two mechanisms": the
   exceptions are never random, they are the pages one template variant
   or one URL pattern produces.

Then find the pages **outside** the template. Read
`summary.totalInternalIndexableUrls` from the Crawl Overview — after a deploy
that canonicalises everything to another host it should be near zero, and
each remaining indexable page is either a self-referencing exception or a
page with no canonical at all. `Canonicals: Missing` is usually tiny and safe
to read back whole, and it names the second kind. Reconcile: canonicalised +
self-referencing + missing should account for every internal HTML URL, and
the residual is the group you have not characterised (see "Reconciling to a
total" above).

## Reading a stored crawl's API date range and filters

When GA4 or Search Console were connected at crawl time, `Internal > HTML`
carries per-URL API fields (GA4 sessions, key events and revenue; GSC
clicks, impressions and position). **Nothing in any export states the date
range the pull covered** — a URL with zero sessions means zero *in some
window*, and a zero-traffic reading without the window is not a finding.
Ask the user first; when nobody remembers, the crawl's own saved
configuration is the first-party source, and it is readable.

Provenance for everything below: observed on one stored crawl under 24.x
and not re-run on 25; the byte layout is Java serialisation and may shift
between versions, so verify the pattern against a known date before
trusting a scan. On 25, try `sf_get_config_spider(category: "ga4Config")`
and `"gscConfig"` first: loading a crawl loads the configuration it ran
with (the vendor says so for segments), which may make the byte scan
unnecessary — unexercised.

The saved config lives outside the MCP's base directory, at
`~/.ScreamingFrogSEOSpider/ProjectInstanceData/<crawl id>/serialised_data/spiderconfig`
(`<crawl id>` is the listing's `crawlId`; on two crawls run by 25.1 the
`serialised_data` folder held no file named `spiderconfig`, so list that
folder first and expect the file name to differ), and a script run with
`sf_run_node_script` can read it — the runner's filesystem reach is not
confined (see `tool-surface.md`). It is a Java-serialised object graph, not
JSON, so the recipe is a byte scan rather than a parse:

- The GA4 and GSC ranges sit in `seo.spider.config.DateRangeConfig`
  objects whose `mStartDate` / `mEndDate` are `java.time.LocalDate`. A
  `LocalDate` serialises through `java.time.Ser` as a 9-byte block:
  `77 07 03`, then the year as a big-endian int32, then one byte each for
  month and day. Scan the file for `77 07 03` and decode the six bytes that
  follow. An epoch-millisecond scan finds nothing, which reads as "dates
  not stored" — they are stored, just not as timestamps.
- A relative window appears as a `GoogleRelativeDateRange` enum string
  (e.g. `PAST_30_DAYS`) near the same block — read it as text with a
  `latin1` decode of the buffer and a regex.
- The GSC block also carries the country filter
  (`Iso31661Alpha3CountryCode`), the device filter and the results type,
  as readable strings; the GA4 block carries the max-results value, the
  row cap that truncates a wider pull (below).

**After any API re-pull over a wider window, run the per-URL superset
check before using the data.** A 12-month pull must contain every URL that
had traffic in the 30-day pull — the wider window is a superset by
definition. The failure shape: one source obeys this and the other does
not — the count of URLs with sessions *falls* after the wider pull, and a
share of the URLs with recent sessions comes back null for the wider
window, concentrated in one path prefix and at every traffic level,
including well-trafficked pages, while headline totals rise by a
plausible multiple and hide it entirely. The test is: export
`Address` plus the API fields before and after, join on `Address`, and
list URLs > 0 in the narrow pull that are null or 0 in the wide one. A
non-empty list means the wide pull is truncated (the GA4 max-results cap
is the leading suspect) or otherwise broken; do not use it. And in either
pull, **treat null as unknown, never as zero** — a URL absent from the API
response is not a URL with no traffic.

## Confirming a single claim cheaply

Before asserting something about one URL, `sf_url_info` on it. It returns
every stored field in one call, so it settles status, indexability, inlink
count and crawl depth at once — far cheaper than exporting a filter and
searching it, and it is the right way to sanity-check a claim that came out
of an aggregation before it goes in front of a client or a developer.

Note that the crawl is a point-in-time snapshot of a possibly depth-limited
sample. Counts from it are a **floor**, not a census, and should be described
that way. Component attribution generalises even when counts don't: a
template either emits a crawlable link or it doesn't, regardless of how many
pages the crawl happened to reach.

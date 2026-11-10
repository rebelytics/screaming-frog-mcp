---
name: screaming-frog-mcp
version: 1.1.0
core_max_lines: 598
description: "Companion skill for the official Screaming Frog SEO Spider MCP server (SEO Spider 25), which drives a desktop SEO Spider instance via sf_* tools (sf_list_crawls, sf_load_crawl, sf_task_status, sf_export_seo_element_urls, sf_set_config_spider, sf_crawl_from_list). Load whenever the user works with Screaming Frog through an MCP: analysing a saved crawl, exporting an SEO element or report, starting or configuring a spider or list crawl, pulling inlinks or outlinks, checking indexability, robots.txt blocking, canonicals, directives or response codes, or attributing crawl findings to the templates that emit them. Also load on any sf_ tool name — even when the MCP isn't named but crawl data is clearly needed. Read this BEFORE the first sf_ call: a client still holding the previous release's tool list loses the loaded crawl on its first call, exports run as background tasks whose result is fetched separately, and a paused or partial crawl needs handling rather than rejecting."
license: CC-BY-4.0
metadata:
  author: Eoghan Henn and contributors
  source: https://github.com/rebelytics/screaming-frog-mcp
---

# Screaming Frog SEO Spider MCP — Companion

A field guide to how the official Screaming Frog SEO Spider MCP server
actually behaves. It documents the parameter rules, response shapes, cost
traps and boundaries that the tool descriptions leave out, so a session
using it is structured and avoids the mistakes that are expensive to
discover in flight. It does not replace the server's own tool descriptions;
it adds the operational knowledge those descriptions omit.

> Tool names below are written without the server prefix (e.g. `sf_load_crawl`).
> Your environment prefixes them — match by the suffix.

**This guide describes the server that ships with SEO Spider 25** (verified
on 25.1). The release renamed six tools, added about twenty, and moved loads
and exports to background tasks, so the first step of every session is to
establish which release each side is on — see the session-start protocol.
Behaviour carried over from 24.x and not re-run since is marked where it
appears.

---

## The mental model

This MCP is a **remote control for a desktop application**, not a database.
Three consequences shape everything else:

1. **It holds one crawl at a time.** Every query runs against whatever crawl
   is currently loaded. Nothing is scoped per-call.
2. **It runs on the SEO Spider's machine, which may not be yours.** Its
   file tools read and write a fixed base directory there. Where your agent
   runs on another machine (a cloud session), that directory is reachable
   only through the MCP's file tools; on the same machine your shell can
   read it directly, though the file-tool loop is still cheaper on context.
   A script run via `sf_run_node_script` can write outside the base
   directory: the cheapest bulk handoff. See `references/tool-surface.md`.
3. **It answers with whole reports.** A single export can be hundreds of
   thousands of rows. There is no aggregation, no `count`, no `GROUP BY`.
   You have to impose that yourself, and doing it wrong is what makes
   sessions expensive.
4. **The call is not the work.** Loads and exports start a background task
   and return at once; the outcome — the count, the saved path, the empty
   signal — is whatever `sf_task_status` says afterwards, and other tools
   are refused until the task ends.

Internalise those four and the rest of this skill is mostly detail.

### Sharing one instance between sessions

The first consequence has a corollary the tool descriptions never draw:
**loaded state is global across every MCP client driving the instance.** The
unit of concurrency is the instance, not the session. Two sessions reading
the same loaded crawl through exports and lookups is fine; a state-changing
call from either one silently switches the data under the other, and the
other session cannot tell — no query names the crawl it ran against.

The state-changing calls are `sf_load_crawl`, `sf_clear_crawl`, `sf_crawl`,
`sf_crawl_from_list`, `sf_set_crawl_mode` and `sf_resume_crawl` (on a paused
crawl the risky call is *resume*, not load — resume starts fetching the
queue under everyone). Three more change things nobody thinks of as state:

- **`sf_set_config_spider` changes the instance's configuration**, which
  outlives the loaded crawl and applies to the next crawl any session starts
  — a URL limit or a rendering mode left behind is someone else's surprise.
  Read the value first and restore it.
- **A running task or crawl blocks the instance.** While one session's
  export runs, every other session's calls are refused with a message naming
  the task; `sf_cancel_task` from any session stops it.
- **A call to a tool name the server does not have drops the loaded crawl**
  for every session (see `references/gotchas.md`, `Unknown tool`).

On 24.x a heavy export that timed out also dropped the loaded crawl
(observed on a `Links: All Inlinks` bulk export of a ~100k-URL crawl). On 25
the same export on a 10k-URL crawl finished in seconds as a background task
and the crawl stayed loaded; an export that runs for minutes has not been
tested, so check `sf_crawl_progress` after any long one.

Working rules: give each session its own export subfolder (the base
directory is shared and `sf_write_text_file` overwrites silently); announce
any state-changing call to the other session, or to the user, before
making it; and before `sf_clear_crawl` or any handover to the desktop UI,
copy working files still needed out of the base directory with a Node
script — the only routes back to them run through the MCP, and the script
runner is gated on a loaded crawl the UI now holds. Whether two sessions
can queue exports against each other without corrupting anything is
untested.

---

## Session-start protocol

**Check the surface before the first call.** Two releases can meet in one
session: an application upgraded while the MCP client stayed open leaves the
client announcing the old tool list to a new server. Read your own tool
list, without calling anything: `sf_task_status` and `sf_set_config_spider`
present means the 25 surface. If they are absent while `sf_list_crawls`
returns `crawlId` (not `instanceDirName`), the client is stale — stop and
tell the user it needs a restart or a new session. **Never probe with a
remembered tool name:** on 25 an unknown name returns `Unknown tool` and
drops the loaded crawl (`references/gotchas.md`).

**Then load a crawl.** Every query tool fails with
`You need to load or run a crawl first` until you do.

This is not just a first-session step. **The loaded crawl does not survive a
restart of the host application** — tools that worked minutes earlier start
failing after the user restarts, and the error looks like a broken server
rather than lost state. If a previously-working `sf_*` call suddenly returns
that error, re-run `sf_load_crawl` before assuming anything is wrong.

```
sf_crawl_progress()               → what is loaded now (crawlId, stateName)
sf_list_crawls(limit: 10-15)      → pick the crawl
sf_load_crawl(crawl_id: "<crawlId>")
sf_task_status()                  → SUCCEEDED / "The crawl has been loaded successfully"
```

The `crawl_id` is the `crawlId` field, not the `crawlName`. The load is a
background task: the call returning says only that it started.

**Loading a crawl stored by an older version upgrades it in place.** A crawl
last opened on 24.x went through a backup phase and a database conversion
on its first 25 load (about 70 s for 10k URLs) and was listed with the new
`projectVersion` afterwards. Whether the older application can still open it
was not tested, so say so before loading a pre-25 crawl the user may want to
reopen elsewhere.

**Optional, off unless the user asks for it: a release check.** The server
exposes no application version, and running headless it shows no update
notice — a major release can go unnoticed for weeks. The highest
`projectVersion` in `sf_list_crawls` is a floor for the installed version
(it is the version that last opened that crawl). Where the user has opted
in and the session has network, compare it once per day with the vendor's
release-history page and say in one line when a newer release exists and
whether its notes mention the MCP server; never block the task on it. This
guide's own claims were verified on 25.1, so a newer major release also
means this guide may be behind.

**Record the crawl's vintage at load time, and carry it into every
deliverable.** A saved crawl is a snapshot, and `sf_list_crawls` gives you its
date for free at the exact moment you are already looking — so there is no
cost to capturing it, and no excuse for a finding that cannot say how old its
evidence is. The distinction that matters downstream: structural conclusions
survive staleness (a template emits no canonical; a section is reachable only
through pagination; a class of URLs carries the wrong directive), while
**per-URL lists do not** — any list of specific URLs derived from a crawl is a
claim about the state of those URLs on the crawl date, not today. Before
shipping one, either re-check a stratified sample live, or state the crawl
date in the deliverable and say plainly that the list is as-of that date. A
per-URL list shipped without its vintage invites the recipient to act on rows
that have since been fixed, redirected or removed, and every such row costs
credibility that the structural finding underneath it did nothing to deserve.

**Check what is loaded before loading anything.** `sf_crawl_progress` is
read-only (refused only while a background task runs): it reports the
`crawlId` of the crawl currently in memory plus `completed`, `waiting`,
`stateName` and `crawlRunning`. If the crawl you want is
already loaded, you don't need `sf_load_crawl` at all — and if the app holds a
**paused** crawl, `sf_load_crawl` refused outright on 24.x (not re-run on 25) with
`IllegalStateException: UNEXPECTED_MESSAGE loadCrawl in state: SpiderPausedState`.
That refusal is not a problem to route around: query the paused crawl in place.
Resuming it to make a load succeed would destroy someone's work.

**Clearing it, though, is the documented handover path — if the crawl has a
durable home.** A paused crawl stored as a database project survives
`sf_clear_crawl`: the call empties the MCP instance's memory and leaves the
project in `sf_list_crawls` with every URL intact and immediately reloadable.
Since `sf_load_crawl` was refused in `SpiderPausedState` on 24.x, this is the only
MCP-side lever for handing the instance back to the desktop UI. Establish
durability first (the crawl must appear in `sf_list_crawls` as a stored
project), then re-check `sf_list_crawls` afterwards rather than trusting the
storage model. For a crawl with no durable home, clearing still destroys it.

**Handing over from the desktop UI is gated, and progress is mis-reported
while it runs** (24.x observations, not re-run on 25). A crawl open in the desktop UI stays locked to that
instance — `sf_load_crawl` fails with `Crawl is open in another SEO Spider
instance` until the user closes it in the UI, so ask them to close it up
front rather than rediscovering the lock on every handover. And
`sf_crawl_progress` is scoped to the MCP's own instance: while the user's
crawl runs in the UI it keeps reporting the MCP's previous crawl (e.g.
`completed: 283, percentComplete: 100`) — a confident status about the
wrong job. For any crawl started outside the MCP's instance, poll
`sf_list_crawls` and judge by `urlsCrawled` on the new crawl's id.

**`percentageComplete` is real progress — but a low number does not mean an
unusable crawl.** A crawl reading `7.0` genuinely has most of the site
un-crawled, and `sf_crawl_progress` will show the queue behind it (in one case
62,551 completed against 785,256 waiting). The mistake to avoid is the
inference, not the reading: a deliberately-sampled or paused crawl is usually
exactly the one the user means by "the last crawl", and its **component-level
findings generalise even though its counts are floors**. Judge usability by
`urlsCrawled` and `spiderMode`; use the queue depth to decide how to caveat
the numbers, not whether to trust the crawl.

**The converse also holds: a high `percentageComplete` can hide a non-empty
queue.** The listing's figure is a rounded ratio — a paused ~100k-URL crawl
with a hundred-odd URLs still waiting reads `100.0`. Exports hold fetched
URLs; the queue holds discovered-but-unfetched ones, and it is the tail of
discovery, so it concentrates the lightly-linked URLs an orphan or "not
linked" analysis is hunting for. Before any absence-based conclusion, read
`waiting` from `sf_crawl_progress`; if non-zero, export `Queued URLs` (a
small queue exports fast even when heavier exports time out) and remove its
rows from the absent set — "Absence and subtraction traps" in
`references/workflows.md` has the recipe and its two siblings.

`spiderMode` distinguishes `Spider` from `List` crawls, which matters when
the user refers to "the control crawl" among several.

**Crawl and project names are human-authored claims, not behaviour.** A
`crawlName` saying "full crawl" describes intent at naming time — the crawl
behind it may be sampled, paused, or from a different scope than the name
implies, and names are the artefact least likely to be updated when the
crawl's reality changes. Judge a crawl by `urlsCrawled`, `spiderMode` and
its timestamps, never by its name; the same claims-vs-behaviour split
applies to any label travelling alongside data on this server, and where
the two disagree, the data wins.

---

## Getting data out without flooding the context

Every export writes to a file you name — `file_path` is required, and a
call without it fails — and runs as a background task. The call's return
value says only that the task started. Fetch the result:

```
sf_export_seo_element_urls(..., file_path: "wk/blocked.ndjson")
sf_task_status()
```

```json
{"format":"NDJSON","path":"/…/wk/blocked.ndjson","total_records":2409,"sample":"{…one row…}"}
```

Then write a Node script with `sf_write_text_file` and run it with
`sf_run_node_script` to aggregate on the user's machine, returning only the
summary you actually want to read. The full loop is in
`references/workflows.md`.

Most of that is what a careful operator would do anyway. These details are
not obvious and are worth carrying:

- **The receipt's `total_records` is the cheapest count the server offers.**
  One line answers "how many?" without touching a single row. Element
  exports, tabular reports and single-file bulk exports carry it; the Crawl
  Overview report and multi-file exports do not.
- **`sf_task_status` describes the last task that started, not your last
  call.** A call that fails synchronously starts no task, so the status
  still shows the export before it — count included. Check that the path in
  the result is the file you just asked for before reading a number off it.
- **An empty result has two signatures, and can be a lie.** An empty filter
  fails the call itself: `No URLs match filter '…' …, so there is nothing to
  export`. An empty bulk export *succeeds*: the task ends `SUCCEEDED` with
  `Report generated no data. No file was written to …`. Both mean the
  selection is empty, not that the thing doesn't exist — crawl configuration
  decides whether many filters get populated at all. Cross-check with
  `sf_generate_report(category: "Crawl Overview")` before reporting zero. A
  non-zero Crawl Overview count alongside an empty row export is not a
  contradiction to debug: the summary is computed independently of the
  display layer, so a display-config setting can suppress the rows while
  the summary counts them — **the summary is the count to trust**. See
  `references/gotchas.md`.
- **Never use `max_rows` when you want a count.** It caps `total_records`,
  so `max_rows: 1` reports `total_records: 1` against a 2,409-row set. It
  looks like an answer and is an artefact of your own parameter. Probe with
  a full export, not a capped one.
- **The result's shape depends on the tool.** For element exports the
  receipt's `path` is the file; for reports and bulk exports `path` is a
  temp file and the real location is the trailing `Export saved to …`
  line; and the `sample` is one whole row, which on a page-content export
  is an entire page arriving in the status result. The table is in
  `references/gotchas.md`.

## Six rules for querying something you cannot see inside

These are not Screaming Frog rules — they hold for any tool server whose
internals you can only infer from responses. Each bites in a specific way
here.

- **Run the control call before reading anything into an absence.** An empty
  result and a data source that never held the thing are indistinguishable
  from the response alone. Here the control is
  `sf_generate_report(category: "Crawl Overview")` plus a check that the
  parent population is non-empty: a crawl holding zero JS URLs cannot report
  a blocked script whatever the truth. Report "there are none" and "the crawl
  cannot tell us" as the different answers they are. Sharpened form: the
  strongest control is a **positive control — the identical call against a
  population known to hold data** — not merely an unscoped call, which can
  succeed on other data and disguise the gap. A crawl listed in
  `sf_list_crawls` is a registry entry, not proof its data is loaded and
  queryable, and a crawl that finished (or was handed over) within hours is
  the leading suspect when everything comes back empty. Read the zeros'
  distribution: empty across every report means the scope or coverage is
  the problem; empty in one filter means that filter selected nothing.
- **Inspect the shape of what came back before scripting against it.** This
  server has at least three file-output formats and they do not follow the
  server, they follow the tool: element exports are NDJSON, `sf_generate_report`
  writes one pretty-printed JSON document, `sf_url_links` writes a bare URL
  list. **The extension you chose yourself is not evidence of the format
  inside.** Read the first lines back (`sf_read_text_file` with a small
  `limit`) before writing a parser, or have the script sniff the first
  character.
- **A filtered result identical to the unfiltered one means the filter did not
  apply.** If a call scoped by `segment_name`, `links_category` or a filter
  returns the same `total_records` as the unscoped call, treat the parameter
  as ignored rather than concluding the segment is the whole. The same tell
  applies to a `data_fields` list that comes back with every field present.
- **Verify a count against an independent derivation.** The receipt's
  `total_records`, the Crawl Overview totals, and a line count inside the
  aggregation script should agree. When they don't, the cause is usually a
  parameter of your own — `max_rows` caps `total_records`, so a capped export
  reports your parameter back as if it were data — or a status that belongs
  to an earlier task.

- **A scope you did not set is still a scope, and it appears in neither the
  request nor the response.** This server makes the point harder than most:
  the loaded crawl is chosen out of band by `sf_load_crawl` and is named
  nowhere in a query's parameters or its results, and the crawl's own
  configuration silently decides which filters can hold anything at all. Add
  to that the caps you set yourself — `max_rows` on an export, `limit` on
  `sf_list_crawls`, a `crawlTotalLimit` left in the instance by an earlier
  crawl — and a total can be three defaults deep without one of
  them being visible. A default cannot be found by re-reading the call (it
  isn't there) or by inspecting the result (it isn't there either); the only
  detector is varying the parameter deliberately, so run the export twice
  with the cap at its extremes before quoting any total, and state the crawl
  and the config assumptions beside the number. Treat `total_records` and an
  export that ran to completion as reports on what you **asked for**, never
  on what the crawl holds — a finished export supplies the feeling of
  completeness that would otherwise prompt the check. A per-row value that
  looks like history (a timestamp, a `Last Modified` header) says nothing
  about how wide the scope was.
  And a scope you *did* set is still an aggregate that says nothing about
  now — a distinct failure from the unset default above, because here the
  crawl was chosen deliberately and named in the deliverable, and the figure
  is still not evidence about the present. Every number this server returns
  is an as-of measurement of one crawl: a 4xx share, the proportion of a
  pagetype missing a canonical, an indexability rate. Each answers "how many,
  when that crawl ran" and is read as "how many now", and the two diverge
  exactly when the thing has been fixed since — the case where acting on the
  number is most wasteful, because the remediation ticket goes out for a
  template that was corrected a fortnight ago. **So before a rate carries a
  recommendation, band it.** Run the identical count across the stored crawls
  (`sf_list_crawls` → `sf_load_crawl` per crawl, same report, same filter,
  same caps) and read the sequence rather than the single value; where the
  data source takes a since-date rather than a snapshot handle, the same
  move is nested windows differenced against each other, one call per band.
  Then confirm the most recent band live — `sf_url_info` over a stratified
  sample, or the re-check workflow in `references/workflows.md` — which is
  what "still happening" actually requires whenever the newest crawl is not
  fresh. Apply the positive control from the first rule to that newest crawl
  before believing it: an unloaded or empty crawl reports zero and reads as
  "fixed". Two things belong in the deliverable: which crawl the finding
  actually lives in, and whether it is still present in the most recent
  evidence you hold. The sequence also recovers the mechanism a lone total
  destroys — a count non-zero in the older crawls and exactly zero in every
  crawl since some date names both the cause and roughly when it stopped.
  Two riders: check the denominator, because a dramatic percentage over a
  handful of URLs is not a finding; and where a report attaches a headline
  rate to a named page or example URL, read it as scoped to that row until
  proven otherwise, not as a crawl-wide total that happens to be displayed
  beside an illustration.
- **Test an identifier before using it as a join key across two exports.**
  Crawl exports carry no durable row id — identity is the `Address` column,
  and row order is an artefact of the export, not of the data. So a
  before/after comparison against a control crawl must be joined on
  `Address` (or the URL pair on a link export) and diffed field by field;
  inferring change from row counts, row order, or set membership over
  anything else measures the instrument. The failure is quiet in both
  directions — an unstable key manufactures a difference in exactly the
  operation used to detect difference, and had something genuinely changed,
  the same diff would have been believed for the wrong reason.

The seventh rule of this family — keep full-payload views small and prefer
count views — is the section immediately above, because on this server it is
the single most expensive mistake available.

## Reference files — load on demand

- **`references/tool-surface.md`** — every tool, grouped by job, with what
  each is for and which are verified on 25, carried from 24.x or
  unexercised, plus the six renames. Read when choosing a tool or when you
  need to know whether a capability exists at all. Also holds the rules for
  starting crawls: crawl mode and `sf_crawl_from_list`, the authentication
  routes, the `config_path` provenance warning, the configuration tools
  (URL limit, rendering, Store HTML, custom extraction, segments) and how
  to bound a crawl — read them before crawling any auth-protected,
  config-driven or large target, and before changing any setting.
- **`references/workflows.md`** — the export→aggregate→summarise loop in
  full, plus the recipes that turn crawl reports into findings: component
  attribution from inlink exports, robots-rule attribution, reconciling
  buckets to a total, re-checking a URL list on the user's machine (a
  List-mode crawl through the SEO Spider's own CLI spawned from the script
  runner, which can also land page text on disk, or a detached status
  sweep), the absence and subtraction traps (queued URLs, snapshot-vs-live
  joins, self-links), canonical verification across a template, and reading
  a stored crawl's API date range. Read before any multi-step analysis,
  and before crawling a list of URLs.
- **`references/gotchas.md`** — the parameter rules and boundaries that
  produce errors or silent wrong answers, including the ones with no
  workaround, the refusals a running crawl or task produces, and the task
  result shapes per export family. Read before the first export, and
  whenever a call errors unexpectedly.

---

## Choosing what to ask for

The server organises data as **SEO element × filter**, mirroring the desktop
UI's tabs and filters. Discovery is cheap and the names are not guessable, so
resolve them rather than assuming:

```
sf_list_filters_for_seo_element(seo_element_name: "Response Codes")
sf_list_fields_for_seo_element_filter(...)
```

**Data-field names are filter-specific, not global.** A field valid for one
filter is rejected for another — `Unique Inlinks` exists on `Internal`/`HTML`
but not on `Response Codes`/`Internal Blocked by Robots.txt`. The error
helpfully enumerates the valid set, so a wrong guess costs one round trip
rather than a wrong answer, but checking first is cheaper.

Three limits of those listings, worth carrying:

- **They prove a filter or field is addressable, not populated or
  current.** The catalogue is an accretion mirroring everything the
  desktop app can display, so a filter can be listed and hold nothing —
  either because the crawl config never collected it (the known case) or
  because the surface is simply unused on this site. A zero-row probe on
  an advertised filter has three readings — none exist, never collected,
  or not in use — and the server cannot distinguish them; only whoever
  configured the crawl can. Report an unpopulated advertised filter as an
  observation with a question attached ("this returns nothing — is it
  expected to?"), never as a defect finding; where a populated sibling
  filter covers the same concept, unused is the leading hypothesis, not
  broken.
- **A name that validates can still fail at execution, and the failure's
  blast radius misdirects.** The name-validation errors above are the
  cheap, informative half — they enumerate the schema. Execution-side
  failures are the misdirecting half: a call that silently drops the
  loaded crawl (an unknown tool name does) makes the *next*, unrelated
  call fail, so the error arrives attached to the wrong artefact. When a failure is larger than
  the thing you changed, suspect your own parameter or a state change
  before suspecting the server.
- **Before engineering a workaround for something the export surface
  can't express, enumerate what the server precomputes.**
  `sf_list_reports` and `sf_list_bulk_exports` list
  derived reports the desktop app computes for itself — and the Crawl
  Overview `summary` block reconciles whole issue classes in one read.
  A precomputed report carries the vendor's definition of the predicate
  and is computed per crawl, so it is comparable across crawls; an
  ad-hoc export-and-filter reconstruction is neither. Ask "has the app
  already answered this?" before building the clever route — the
  workaround is the interesting option, which is exactly why attention
  flows to it.

Two fields repay knowing about because they turn a list of problems into a
list of causes:

- **`Matched Robots.txt Line`** (on the blocked-by-robots filters) names the
  exact robots.txt rule blocking each URL. This converts "these URLs are
  blocked" into "this rule blocks these URLs" — which is the actionable form,
  and it surfaces over-broad patterns catching things they were never meant
  to.
- **`Link Path` and `Link Position`** (on inlink exports) give the XPath of
  the emitting element and whether it sits in Navigation, Content, Header or
  Footer. This is what lets you attribute a link count to a template
  component rather than reporting a number nobody can act on. See
  `references/workflows.md`.

---

## Read the aggregate for rules catching more than intended

Once a report is grouped — by robots.txt rule, by emitting component, by
directive — the finding is rarely the biggest number. It is the row whose
shape is wrong for what it claims to be.

A robots.txt rule producing far more URLs than its author would expect, or
matching URLs of a type it was never written for, is an over-broad pattern.
Robots patterns are prefix matches with no implicit end anchor, so an
unanchored path rule also matches content URLs whose slug merely begins with
the same word — which silently blocks real pages. In one crawl, a rule written
for search endpoints was also blocking two live product pages carrying 1,023
internal links between them, and nothing in the totals hinted at it.

The same reading applies elsewhere: a component emitting links at a
non-integer rate per page is per-item rather than chrome; a large link count
resolving to a handful of destinations means an enumerable set, which changes
the question from "stop linking to these" to "why are these not legitimate
targets".

**When you suspect an over-broad rule, test the replacement before
recommending it.** A short matcher implementing `*` and `$` semantics, run
over the exported addresses, gives you still-blocked / newly-released /
newly-caught. That turns a recommendation into a measurement, and it is the
step that separates a useful crawl analysis from a competent one.

## Scope of this skill

This covers **operating the MCP**: the `sf_*` tool surface, parameter rules,
response shapes, cost control, and reading a crawl export for causes rather
than counts. It does **not** cover the SEO Spider desktop application's own UI
and configuration screens, crawl-config design beyond the fields the
configuration tools were exercised with, licensing, or any capability
that has no `sf_*` tool behind it. For how the application itself behaves —
a failed crawl, a setting, a count that does not match — ask the server's
own `sf_check_faq_before_answering` before answering from general knowledge. Turning crawl findings into developer
instructions is a separate concern — if a companion skill for crawl-to-ticket
methodology is available, load it alongside rather than duplicating it here.

Silence in this file is not evidence a capability is absent. This skill is
loaded precisely because the agent doesn't know the server, so an uncovered
surface reads as a missing feature when it may only be an uncovered surface —
check `references/tool-surface.md` and the server's own tool descriptions
before concluding the tool can't do something. And because the server
fronts a desktop application and also runs code on that machine, the
application's own CLI is part of the reachable surface: check it before
declaring a capability missing — see
`references/workflows.md`, "Re-checking a list of URLs".

**Verification status.** Everything stated as behaviour in this skill was
observed directly. The 25 surface — background tasks, crawl mode and list
crawls, the configuration tools, the renamed tools — was exercised on 25.1.
Behaviour first observed on 24.x and not re-run since (paused-crawl
handling, the desktop-UI lock, the authentication routes, the CLI route,
exports that run for minutes) is marked as such where it appears, and part
of the tool surface has not been exercised at all and is marked in
`references/tool-surface.md`. Treat unmarked claims as tested on 25, marked
ones as needing a re-check — and when you exercise an untested or carried
tool, that is worth reporting back so this guide can absorb it.

---

## Corrected claims

The body above states the current method, undated. Earlier revisions of
this guide carried claims that direct observation later overturned; they
are kept here, collapsed, so a reader who meets one in an older copy or in
notes derived from it can recognise it as retired rather than re-test it.

<details><summary>Old patterns</summary>

- An empty filter proves zero. Retired: crawl configuration decides whether
  a filter is populated at all ("Show external URLs blocked by robots.txt"
  is off by default; resource filters are empty when resource crawling is
  off), so the Crawl Overview report is the population cross-check.
- A low `percentageComplete` is a display quirk. Retired: it is real
  progress — and a rounded 100.0 can still hide a non-empty queue.
- `max_rows` can be used to count. Retired: it caps `total_records`, so the
  receipt reports what was asked for, not the population.
- Pausing a crawl frees the instance for a handover. Retired: pausing never
  frees it; `sf_clear_crawl` is the safe handover path for a
  database-stored crawl.
- The server has no list-mode crawl. Retired: since 25 `sf_set_crawl_mode`
  and `sf_crawl_from_list` run one. The 24.x route — the script runner
  spawning the SEO Spider's own CLI with `--crawl-list` — remains as the
  fallback (`references/workflows.md`, "Re-checking a list of URLs").
- Exports return NDJSON inline when `file_path` is omitted, and the export
  call returns the receipt. Retired with 25: `file_path` is required, the
  call returns a task notice, and the receipt comes from `sf_task_status`.
- An empty filter raises `IllegalArgumentException: start_index is greater
  than the total number of URLs of 0`. Retired with 25: the call fails with
  `No URLs match filter …`.
- The crawl id is `instanceDirName`, and `sf_crawl_progress` names the
  loaded crawl in `databaseId`. Retired with 25: both are `crawlId`.
- Six tool names, among them `sf_run_node_js_script` and the
  `sf_list_available_*` family. Retired with 25: see the rename table in
  `references/tool-surface.md`; calling an old name drops the loaded crawl.
- The configuration cannot be read or changed over MCP, so an MCP-started
  crawl has no URL cap you can see. Retired with 25: `sf_get_config_spider`
  shows `crawlTotalLimit` and `sf_set_config_spider` sets it.
- Only one config file can reach a crawl. Retired with 25: both crawl tools
  take `auth_config_path` beside `config_path` (not yet exercised).
- `sf_url_content` takes `show_visible_content_for_html_url`. Retired with
  25: `page_content_type` (`RAW_HTML`, `VISIBLE_TEXT`, `MARKDOWN`).

</details>

## Feedback on this skill

If the user has methodology feedback, offer to draft a report for
github.com/rebelytics/screaming-frog-mcp, running the feedback pre-flight
first; if the problem is the agent not following the skill's rules,
acknowledge and correct it instead.

Screaming Frog SEO Spider MCP Companion: Eoghan Henn and contributors | CC BY 4.0

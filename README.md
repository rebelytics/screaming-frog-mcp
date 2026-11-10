# screaming-frog-mcp

Open-source companion skill for the **official Screaming Frog SEO Spider MCP server** (SEO Spider 25), which drives a desktop SEO Spider instance via `sf_*` tools. A field guide to how the server actually behaves: the parameter rules, response shapes, cost traps and boundaries that the tool descriptions leave out, so a session using it is structured and avoids the mistakes that are expensive to discover in flight. Load it into any MCP-capable AI agent (Claude, Cursor, Codex, n8n, etc.).

## When it triggers

Whenever the user works with Screaming Frog through an MCP: analysing a saved crawl, exporting an SEO element or report, starting or configuring a spider or list crawl, pulling inlinks or outlinks, checking indexability, robots.txt blocking, canonicals, directives or response codes, or attributing crawl findings to the templates that emit them. Also on any `sf_` tool name — even when the MCP isn't named but crawl data is clearly needed. Read it before the first `sf_` call: a client still holding the previous release's tool list loses the loaded crawl on its first call, exports run as background tasks whose result is fetched separately, and a paused or partial crawl needs handling rather than rejecting.

## What this repo contains

- `SKILL.md` — the mental model (one crawl at a time, on the user's machine, answered with whole reports, through background tasks), the session-start protocol with its release check, getting data out without flooding the context, the six rules for querying something you cannot see inside, choosing what to ask for, reading the aggregate for over-broad rules, the scope of the skill, and the pointer list for the reference files.
- `references/tool-surface.md` — every tool, grouped by job, with what each is for and which are verified on 25, carried from 24.x or unexercised; the six renames; crawl mode and list crawls, the configuration tools, and the rules for starting and bounding a crawl.
- `references/workflows.md` — the export→status→aggregate→summarise loop in full, plus the recipes that turn crawl reports into findings: component attribution, robots-rule attribution, reconciling buckets to a total, crawling or re-checking a URL list on the user's machine, the absence and subtraction traps, canonical verification, and reading a stored crawl's API date range.
- `references/gotchas.md` — the parameter rules and boundaries that produce errors or silent wrong answers, including the ones with no workaround, the refusals a running crawl or task produces, and the task result shapes per export family.
- `references/ceilings.txt` — line ceilings for the reference files, read by the skill validator.
- `LICENSE` — CC BY 4.0.

## Install

Install the whole skill directory — `SKILL.md` and `references/` must travel together — into your MCP-capable agent's skills directory, following the client-specific path, and restart the client.

## Contributing

Open an issue or a PR at [github.com/rebelytics/screaming-frog-mcp](https://github.com/rebelytics/screaming-frog-mcp).

## Credits

- Original author: [Eoghan Henn](https://www.rebelytics.com) / [LinkedIn](https://www.linkedin.com/in/eoghanhenn)

## License

CC BY 4.0 — the licence text is in [LICENSE](./LICENSE): the full legal code, unmodified, as Creative Commons publishes it; the [human-readable summary](https://creativecommons.org/licenses/by/4.0/) says what it allows. Use it, fork it, adapt it, monetise it. Keep the attribution.

Copyright (c) 2026 Eoghan Henn / rebelytics.com

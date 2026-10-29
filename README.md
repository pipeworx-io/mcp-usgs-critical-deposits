# @pipeworx/usgs-critical-deposits

USGS's "Critical mineral deposits of the United States" (ver. 3.0, September
2026) — 1,271 documented mineral deposits across 41 states and Puerto Rico,
each classified by mineral system and deposit type, with production history,
resource/reserve figures and references, compiled from MRDS, ARDF and USMIN
plus industry reports and the literature. Distinct from `usgs-mrdata` (raw
MRDS occurrences, no curated critical-mineral resource/reference compilation)
and `usgs-mineral-commodities` (national production/trade summaries, no
per-deposit records).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `usgs_cmd_deposits_search(commodity?, state?, bbox?, limit?)` — search by
  critical mineral/commodity (e.g. "lithium", "cobalt", "ree"), US state or
  territory abbreviation, and/or a bounding box. Returns each deposit's id,
  name, state, coordinates, minerals, deposit type and resource figures. An
  unrecognized commodity returns `found: false` plus the full list of
  commodities this dataset actually has deposits for (see "Commodity
  vocabulary" below).
- `usgs_cmd_deposit_detail(id)` — one deposit's full record: minerals,
  deposit type, mineral system, production history, resource/reserve
  figures, references, a link to the underlying MRDS/ARDF/USMIN record, and
  the release version/DOI.

## Auth

Keyless.

## Data sources

- <https://energy.usgs.gov/arcgis/rest/services/Hosted/Critical_mineral_deposits_of_the_United_States/FeatureServer/0/query>
  — the live ArcGIS FeatureServer this pack proxies per request.
- <https://doi.org/10.5066/P9K1HBNT> — the data release itself (ScienceBase
  item `6464de5bd34ec179a83d9e6c`), "Critical mineral deposits of the United
  States (ver. 3.0, September 2026)". FGDC metadata revision date 2026-09-09;
  USGS's public announcement of the update is dated 2026-09-29.

## Why this proxies live rather than baking a table

The release also ships a download-only Excel/shapefile bundle, but USGS also
stands up the identical table as a keyless ArcGIS FeatureServer (verified live
2026-10-08) — so, unlike `usgs-mineral-commodities` (genuinely file-only),
this is a live-call-per-request proxy, not a mirror, and needs no reuse grant
under root CLAUDE.md's "PROXYING is not REDISTRIBUTING". No KV bake, no
module-scope data table.

## The upstream is measurably flaky

Measured live 2026-10-08: the exact same unmodified query (`where=1=1&
outFields=fid&resultRecordCount=3`) returned HTTP 502 on six consecutive
attempts over ~45 seconds, then HTTP 200 on the next attempt with nothing
changed on this end — this is the upstream ArcGIS hosting infrastructure
hiccuping, not a malformed request. Every call in this pack throws loudly on
a non-2xx status, a non-JSON body, or an ArcGIS-shaped `{"error": {...}}`
response, naming the upstream status — never degrading into `found: false`
or an empty deposits list, which would be indistinguishable from a real
zero-result query. See `src/index.test.ts` for the pinned contract.

## Commodity vocabulary

The `CritMin` field in the underlying dataset is open text, comma-separated,
and mixes individual elements with USGS's own group shorthand ("ree", "pge")
and a few raw multi-word entries ("palladium rhenium", "tantalum tin") — it
is not a clean one-token-per-commodity field, and it is not the same list as
USGS's press-release count of "59 critical minerals" (the 2025 List of
Critical Minerals, 60 commodities, minus metallurgical coal, which this
compilation does not address). The two numbers differ because several of the
60 named commodities have zero documented deposits in this release, and
because grouped tokens (`ree`, `pge`) stand in for multiple individually-named
elements on the official list.

This pack therefore validates `commodity` against the vocabulary **derived
live from the dataset itself** — every distinct comma-split, lower-cased
token actually present across the 1,271 deposits (measured 2026-10-08: 47
tokens) — rather than a retyped copy of the press-release list. That keeps
`deposits_search` from ever saying "not found" for a commodity it would then
also fail to match. The vocabulary is fetched once per isolate and memoized;
a fetch failure clears the memo so the next call retries rather than latching
an empty vocabulary for the isolate's whole life.

## Known field quirks

- `state` is a 2-letter US state/territory abbreviation; Puerto Rico is `PR`.
  There is no country filter — this dataset is US-only (with PR).
- `resources1`–`resources4`, `prduction1`–`prduction3` and `comments1`–`comments3`
  are USGS's own text-overflow continuation columns (a single free-text cell
  that ran past 254 characters spills into the next numbered field) — this
  pack concatenates the non-empty ones into `resources` / `production_history`
  / `comments` arrays rather than exposing the raw numbered fields.
- `references_` (trailing underscore in the raw field name — `references` is
  a reserved-ish word in some ArcGIS client libraries) is exposed as
  `references` in this pack's output.
- Empty cells in the source come back as a single space `" "` or the literal
  string `"--"`, not an empty string or `null` — both are filtered out before
  building the `resources`/`production_history`/`comments` arrays.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "usgs-critical-deposits": {
      "url": "https://gateway.pipeworx.io/usgs-critical-deposits/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/usgs-critical-deposits/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1857+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/usgs_cmd_deposits_search \
  -H 'Content-Type: application/json' \
  -d '{"commodity":"lithium","state":"AR"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/usgs_cmd_deposits_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-usgs-critical-deposits` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/usgs-critical-deposits/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "usgs-critical-deposits": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-usgs-critical-deposits"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Usgs Critical Deposits data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

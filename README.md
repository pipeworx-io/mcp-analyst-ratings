# @pipeworx/analyst-ratings

Today's US stock analyst upgrades, downgrades, coverage initiations and price-target
changes, market-wide, plus the standing consensus rating and covering-brokerage list
for any US ticker.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `analyst_upgrades_today(action?, symbol?, us_only?, limit?)` — every ratings action
  Wall Street brokerages published this session: ticker, company, exchange, brokerage,
  analyst, old and new rating, old and new price target. Answers "which stocks were
  upgraded today", "analyst downgrades today", "today's price target changes", "what
  did analysts initiate coverage on". `action` is one of `all` (default), `upgrade`,
  `downgrade`, `initiated`, `price_target`, `reiterated`.
- `analyst_ratings_for_symbol(symbol)` — consensus rating (Buy/Hold/Sell), the number
  of analysts covering the stock, and the list of brokerage firms publishing
  recommendations on it. Answers "what do analysts think of NVDA", "who covers Apple".

## Auth

Keyless. Both upstreams are public and serve without credentials, registration or a
plan.

## Data sources

- <https://www.marketbeat.com/ratings/> — the combined daily ratings table (~400 rows,
  ~2 MB HTML). Also `/ratings/upgrades/`, `/ratings/downgrades/`, `/ratings/initiations/`,
  which carry the same table filtered to one action and are roughly 10x smaller. The
  pack picks the narrow page when `action` selects one, so the common query does not
  pull 2 MB.
- <https://api.nasdaq.com/api/analyst/{SYMBOL}/ratings> — per-symbol consensus:
  `meanRatingType`, `ratingsSummary`, `brokerNames[]`.

### Traps

- **`api.nasdaq.com` answers `520` to a non-browser User-Agent.** Measured from a
  deployed Cloudflare Worker, 2026-09-10: `PipeworxBot/1.0 (+https://pipeworx.io)`
  → 520 in 65 ms; `Mozilla/5.0 (pipeworx.io)` → 200. The 520 is Cloudflare's generic
  origin error, so it reads as an outage rather than as a refused request. The pack
  sends the `Mozilla/5.0 (pipeworx.io)` form the rest of the catalog already uses.
- **`upgradesDowngrades[]` on the Nasdaq endpoint is always empty.** Checked on AAPL,
  AMGN, NVDA, TSLA and AAL on 2026-09-10 — every one returned `[]`, including tickers
  that were actually downgraded that morning. The field is dead; do not surface it as
  if it carried the day's actions. `analyst_upgrades_today` is where that data lives.
- **Neither source has a date selector.** `https://www.marketbeat.com/ratings/?date=...`
  returns 200 and silently ignores the parameter; `/ratings/2026/09/09/` and
  `/ratings/9/9/2026/` are 404. There is no per-row date in the table either. So both
  tools answer for the current session only, and `as_of` is *derived* — the US/Eastern
  date at fetch time, rolled back off a weekend. The payload says so in `as_of_basis`
  rather than presenting a date the source never gave us. Market holidays are not
  adjusted for.
- **The combined `/ratings/` feed is not US-only.** Measured 2026-09-10: 402 rows
  spanning NYSE 178, NASDAQ 177, TSE 14, OTCMKTS 13, LON 13, NYSEAMERICAN 3, CVE 3.
  `us_only` defaults to true and drops the Toronto/London listings, because the
  question being asked is about US stocks.
- **Parse off `data-clean`, not the cell text.** Every cell carries a
  `data-clean="old|new"` attribute — `AAPL|Apple`, `$445.00|$425.00`, `Buy|Hold` — while
  the rendered text is wrapped in logos, star ratings and upsell links. The action is in
  `data-sort-value` on the second cell. A missing price target is `0`, not an empty
  string, so it has to be mapped to null explicitly or it reads as a $0 target.
- **MarketBeat is reachable from Cloudflare Worker egress** with the honest
  `Mozilla/5.0 (compatible; PipeworxBot/1.0; +https://pipeworx.io)` UA (200, 212 KB,
  187 ms, measured from `wrangler dev --remote` on 2026-09-10). It sits behind
  Cloudflare itself, so this was worth checking before building.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "analyst-ratings": {
      "url": "https://gateway.pipeworx.io/analyst-ratings/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/analyst-ratings/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "analyst-ratings": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-analyst-ratings"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-analyst-ratings
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Analyst Ratings data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

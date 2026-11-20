# @pipeworx/nc-medicaid-rates

North Carolina Medicaid FQHC and rural health clinic (RHC) core-service (T1015) encounter rates, per provider: one rate per NPI and location with PPS or cost-based basis and begin and end dates, from the NC Department of Health and Human Services, Division of Health Benefits.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2015+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `nc_core_rates(program?, provider?, npi?, schedule?, effective_on?, include_history?, limit?)` — per-provider T1015 per-visit rates. Filter by clinic name fragment or 10-digit NPI. `effective_on` returns the span in force on a date. An unmatched name returns the nearest provider names and says what the table does not hold.
- `nc_rate_periods(program?)` — the schedules available (effective date, publication update date, provider counts) and what is not covered.

## Auth

Keyless.

## Data sources

- North Carolina DHHS, Division of Health Benefits (NC Medicaid), FQHC and RHC core-service rate schedules — the per-provider schedules the Division publishes on its FQHC and RHC fee-schedule pages. Public, no account.

Things worth knowing:

- **Rates are per provider, not statewide per code.** Each row is one NPI and location with its own T1015 rate and effective span.
- **The figures are old.** The downloadable schedules are the 2020 (effective 2020-03-31) and 2021 (effective 2021-07-01, updated 2021-10-01) core-service schedules. The Division's later schedules (PPS and alternative payment model rates from 2023) sit in a separate NC fee-schedule portal that this pack does not read, so a provider's current rate may differ. Every response carries `data_as_of` and a `stale_note`.
- The schedules apply to Medicaid Direct (fee-for-service) claims only; managed-care plans pay rates set with NC Medicaid.
- The 2021 rate column is headed "CORE T1015 - Includes Additional 27% Effective" (cut off in the publication); the 2020 schedule states it does not reflect COVID-19 rate increases. The pack returns the rate and does not interpret the heading.
- Provider names are cut off at about 35 characters in the publication; tax id, NPI and location code are returned as strings. There is no address, city or county column, so a place name matches only when it appears inside a provider's name.
- The 2021 schedules include earlier spans (back to 2019) for providers whose rate changed; by default only spans open-ended when published are returned, `include_history` returns all, and `effective_on` picks the span for a date. One RHC row (a Bladen Healthcare location) has a blank PPS/cost-based basis in the publication and is returned with `rate_basis: null`.
- Organisations only: the schedules list health centres and clinics. Any row whose name reads as a person is dropped when the data is read and counted in `individual_rows_dropped` (0 in the current schedules).

## Stored data and refresh

The four schedules are small (about 1,000 rows, 240 KB). They are read from the Division by `scripts/bake-index.mjs` into `src/nc-rates-data.ts`, registered in `workers/gateway/src/pack-baked-indexes.json`; the deploy uploads the table to KV and the gateway injects it on the first call into this pack, so nothing is evaluated at module scope. `node mcps/nc-medicaid-rates/scripts/bake-index.mjs` re-reads the files and rewrites the module in place; it refuses a file that changes layout, shrinks below its floor or carries a non-numeric rate. A newly published schedule is added to the `FILES` list in that script.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "nc-medicaid-rates": {
      "url": "https://gateway.pipeworx.io/nc-medicaid-rates/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/nc-medicaid-rates/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2015+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/nc_core_rates \
  -H 'Content-Type: application/json' \
  -d '{"program":"FQHC","provider":"Anson","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nc_core_rates`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-nc-medicaid-rates` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/nc-medicaid-rates/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "nc-medicaid-rates": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-nc-medicaid-rates"]
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
ask_pipeworx({ question: "your question about Nc Medicaid Rates data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

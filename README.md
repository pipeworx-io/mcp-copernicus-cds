# @pipeworx/copernicus-cds

The Copernicus Climate Data Store (C3S/ECMWF) catalogue — the ~140 published
climate datasets behind ERA5, CMIP6/CORDEX projections, satellite essential
climate variables and seasonal forecasts — plus the ability to submit a
retrieval job against any of them.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1863+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `cds_collections(query?, limit?, offset?)` — list or keyword-search the
  catalogue. Returns collection id, title, first line of the abstract, and the
  temporal/spatial extent. Keyless.
- `cds_collection(collection_id, include_form?)` — one collection in full: STAC
  record (abstract, licence, keywords, extent, documentation links) plus the
  retrieval request form, i.e. the variables, years, levels, area and output
  formats the job will accept. Keyless.
- `cds_submit_request(collection_id, inputs, _apiKey)` — submit an asynchronous
  retrieval job and return the job id. Needs a caller key.
- `cds_job_status(job_id, _apiKey)` — status (accepted, running, successful,
  failed) and timestamps of a submitted job; `found:false` for an unknown id.
- `cds_job_result(job_id, _apiKey)` — the download link with type, size and
  expiry once the job is successful; a job still running returns `ready:false`
  with its status, never an empty result. Same caller key as the submit.

## Auth

Catalogue browsing is keyless and verified so (2026-09-17). Submitting a
retrieval job needs a free CDS Personal Access Token, passed as `_apiKey`.

No platform key is provisioned. The intended env var when one lands is
`PLATFORM_CDS_KEY`; sign up at <https://cds.climate.copernicus.eu>.

`platformKeyEnv` is deliberately NOT declared on this pack's manifest entry
while the key is unset, because `keyBlockedTools()` in the gateway blocks
*every* tool in a pack whose declared platform key has no secret — which would
sink the two keyless catalogue tools that work today. The keyed tool declares
`_apiKey` as required, so routing sinks only that one. Add
`"platformKeyEnv": "PLATFORM_CDS_KEY"` to the pack-manifest entry in the SAME
change that lands the secret.

## Not the same as `copernicus-dataspace`

`copernicus-dataspace` is the EU Copernicus Data Space Ecosystem — the Sentinel
satellite *imagery* catalogue (`catalogue.dataspace.copernicus.eu`). This pack
is the Climate Data Store (`cds.climate.copernicus.eu`): modelled and
satellite-derived *climate variables*. Same programme, different host,
different question.

## Data sources

- <https://cds.climate.copernicus.eu/api/catalogue/v1/collections> — STAC
  collection list. It returns the WHOLE catalogue in one response and **ignores
  `limit` and `q`**, so the keyword filter and paging in `cds_collections` are
  applied here, not upstream.
- <https://cds.climate.copernicus.eu/api/catalogue/v1/collections/{id}> — one
  STAC collection.
- <https://cds.climate.copernicus.eu/api/retrieve/v1/processes/{id}> — the OGC
  API process description. Keyless, and it is where the request form lives;
  the STAC record alone does not say what a retrieval accepts.
- <https://cds.climate.copernicus.eu/api/retrieve/v1/processes/{id}/execute> —
  job submission. Auth is the `PRIVATE-TOKEN` header, not a Bearer token and
  not a query parameter. Unauthenticated it answers
  `401 "authentication required"`; with a bad token,
  `401 "Authentication failed"` — the two are distinguishable.

Accepting the dataset licence on the CDS website is a separate gate from
holding a token: a valid token still gets a 403 for a dataset whose terms the
account has not accepted.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "copernicus-cds": {
      "url": "https://gateway.pipeworx.io/copernicus-cds/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/copernicus-cds/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1863+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/cds_collections \
  -H 'Content-Type: application/json' \
  -d '{"query":"sea ice","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/cds_collections`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-copernicus-cds` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/copernicus-cds/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "copernicus-cds": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-copernicus-cds"]
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
ask_pipeworx({ question: "your question about Copernicus Cds data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

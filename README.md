# @pipeworx/prospeo

Find a person's verified work email by name + company domain, via Prospeo's
person-enrichment API (prospeo.io). Part of the `enrich-waterfall` finder
rotation (fleet #2860).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1745+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `prospeo_find_email(first_name, last_name, domain, company_name?, _apiKey)`
  — returns the verified email only when Prospeo actually reveals it; a
  masked/unrevealed guess is reported as not found.

## Auth

BYO only. Pass your Prospeo API key as `_apiKey`. Get one at
https://app.prospeo.io/api. No platform key — PII vendors are never
platform-keyed (Bruce default D4).

## Data sources

- `https://api.prospeo.io/enrich-person` — person enrichment, used for email
  finding.

## Verification notes (fleet #2860, 2026-10-08, no vendor key held)

- **The old `/email-finder` endpoint is DEAD.** Edge-probed live: HTTP 400
  `{"req_status":false,"error_code":"DEPRECATED","error_toast":"...migrate by
  using our new APIs: https://prospeo.io/api-docs"}`. Several third-party
  write-ups (Apify, Pipedream, a glama.ai MCP server) still document this path
  — do not copy them. The live replacement is `POST /enrich-person`, used
  here.
- **Auth header confirmed live.** `X-KEY: <key>` per
  https://prospeo.io/api-docs/authentication. Edge-probed with no key: HTTP
  400 `{"error":true,"error_code":"INVALID_API_KEY"}` on the real
  `/enrich-person` endpoint — confirms the header name and that the endpoint
  is live and auth-gated. This pack's refusal message says "Prospeo requires
  an API key... get one at app.prospeo.io/api."
- **Response shape is from vendor docs only** (https://prospeo.io/api-docs/enrich-person,
  read 2026-10-08) — we hold no Prospeo key, so no live *successful* call was
  made. The documented example:
  ```json
  { "error": false, "free_enrichment": false,
    "person": { "person_id": "...", "first_name": "Eoghan", "last_name": "Mccabe",
      "email": { "status": "VERIFIED", "revealed": true, "email": "eoghan.*****@intercom.com" } },
    "company": { "company_id": "...", "name": "Intercom" } }
  ```
  Note the doc's own example masks the address (`eoghan.*****@intercom.com`)
  even with `revealed: true` — this pack's parser only accepts an email
  string that does NOT contain `*`, so a masked/teaser address from a live
  call will correctly fall through to `found: false` rather than being handed
  back as a usable contact. **This is the one thing a live key would need to
  confirm**: whether a fully-paid reveal always returns the full address
  un-masked. Flagged as not verified below.
- **Not verified (no key held):** whether a successful paid call returns the
  address unmasked; exact behaviour of `only_verified_email: true` when
  Prospeo has only a guessed (unverified) address.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "prospeo": {
      "url": "https://gateway.pipeworx.io/prospeo/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/prospeo/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1745+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

This pack takes your own API key (`_apiKey`) — we don't front one for it, so there's no curl here that would run without it. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/prospeo_find_email`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "prospeo": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-prospeo"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-prospeo
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Prospeo data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

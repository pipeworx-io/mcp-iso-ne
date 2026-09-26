# @pipeworx/iso-ne

ISO New England market data via ISO Express web services — five-minute LMPs,
system load and generation fuel mix for the six New England states (CT, MA, ME,
NH, RI, VT).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `isone_lmp(location_id?, date?, limit?)` — five-minute real-time LMP with energy,
  congestion and loss components. Omit `date` for the current interval.
- `isone_load(date?, limit?)` — five-minute total system load in MW.
- `isone_fuel_mix(date?, limit?)` — generation by fuel category (gas, nuclear,
  hydro, wind, solar, refuse, wood, coal, oil, landfill gas, imports) with the
  marginal-fuel flag.

## Auth

**BYO only today.** HTTP Basic against `webservices.iso-ne.com`.
Platform fallback env var — **not provisioned**: `PLATFORM_ISONE_KEY`.

Registration is free at
<https://www.iso-ne.com/participate/applications-status-changes/> (an "ISO
Express / Web Services" account), but **ISO-NE approves the account before the
credentials work**. A brand-new registration is refused exactly like a wrong
password, so the pack's 401 message names the approval step rather than implying
the password is wrong.

`_apiKey` accepts either spelling:

- `"username:password"` — plain; the pack base64-encodes it.
- `base64("username:password")` — pre-encoded, same convention as `seo-serp`.

Anything else fails locally with a clear message instead of becoming a bare
upstream 401 that looks like an outage.

**One env var, not `PLATFORM_ISONE_USER` + `PLATFORM_ISONE_PASS`:** the gateway
injects exactly one secret per pack, into `_apiKey`. So the stored credential is
the single string the Basic scheme actually needs. The pack is marked
`byoExpected: true` in `workers/gateway/src/pack-manifest.json` so the vendor-key
ratchet does not fail the deploy; **drop it the day the credential lands.**

## Data sources

- <https://webservices.iso-ne.com/api/v1.1/> — the authenticated API.
  `/fiveminutelmp/current`, `/fiveminutesystemload/current`, `/genfuelmix/current`,
  and `/…/day/YYYYMMDD` for a historical day.
- <https://www.iso-ne.com/isoexpress/> — the public browser UI over the same data.

## Traps

- **There is no keyless fallback. This was checked, not assumed.** Probed
  2026-09-17, all refused:
  - `webservices.iso-ne.com/api/v1.1/fiveminutesystemload/current` → **401**
  - `www.iso-ne.com/ws/wsclient` → **500** (for every parameter shape tried)
  - `www.iso-ne.com/transform/csv/genfuelmix?start=…` → **403**, and it stays 403
    with a browser User-Agent *and* an isoexpress `Referer`. Note it is a 403 and
    not a 404, so the path exists — it is session-gated, not merely UA-gated
    (cf. `reference_gov_site_ua_blocks`, where a UA swap is usually the fix).
- **Response shapes here are UNVERIFIED.** Holding no account, no ISO-NE response
  body in this pack has been observed live. ISO-NE wraps results in a
  singular/plural pair — `{"FiveMinLmps": {"FiveMinLmp": [...]}}` — and the pack
  unwraps that **generically** (descend to the first array-valued property) and
  returns rows as published, without remapping field names. The `envelope` field
  in every response reports which keys it actually walked, so the first real call
  with a credential tells you the true shape rather than hiding it. Tighten once
  credentials exist.
- **Same-day requests can legitimately be empty.** ISO-NE publishes each day's
  reports after the fact. The pack treats a 0-row success as an error and says to
  try the previous day, rather than returning an empty array as a clean answer.
- **ISO-NE does not use a `.json` suffix consistently across its docs**; this pack
  appends `.json` to every path, which is the form the v1.1 API accepts.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "iso-ne": {
      "url": "https://gateway.pipeworx.io/iso-ne/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/iso-ne/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/isone_lmp \
  -H 'Content-Type: application/json' \
  -d '{"limit":100}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/isone_lmp`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "iso-ne": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-iso-ne"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-iso-ne
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Iso Ne data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT

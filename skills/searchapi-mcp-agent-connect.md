---
name: searchapi-mcp-agent-connect
description: >-
  Connect an MCP client to SearchApi's hosted server over OAuth, choose the right tool for
  the job, and understand what the MCP surface does and does not expose.
api: SearchApi hosted MCP server
operations:
  - search
generated: '2026-08-13'
method: generated
source:
  - https://www.searchapi.io/integrations/mcp
  - mcp/searchapi-mcp.yml
  - mcp/searchapi-tool-crosswalk.yml
  - scopes/searchapi-scopes.yml
---

# Connecting an agent to SearchApi over MCP

SearchApi runs a **remote** MCP server. There is no stdio package to install — the GitHub repo
`SearchApi/searchapi-mcp` contains only a README and an MCP registry manifest pointing back at
the hosted endpoint. An MCP client reaches it directly:

```
https://www.searchapi.io/mcp     (streamable HTTP)
```

It is listed in the official MCP registry as `io.searchapi/mcp`, status active.

## 1. Connect

**Claude Code**

```
claude mcp add --transport http searchapi https://www.searchapi.io/mcp
```

**Cursor** — `~/.cursor/mcp.json`

```json
{ "mcpServers": { "searchapi": { "url": "https://www.searchapi.io/mcp" } } }
```

**Claude Desktop** — Customize → Connectors → add the base URL.

## 2. Authorize

Prefer **OAuth**. On first connection a compatible client is redirected through
`https://www.searchapi.io/oauth/authorize`; tokens expire and rotate automatically. The server
publishes RFC 8414 authorization-server metadata and RFC 9728 protected-resource metadata,
supports PKCE (`S256`), and supports dynamic client registration at
`https://www.searchapi.io/oauth/register` — so a client can self-register without a
pre-provisioned client_id.

There is exactly one scope: **`mcp`**. It carries the whole tool surface; there is no per-tool
scoping in OAuth.

The alternative is a static token from the dashboard at
`https://www.searchapi.io/mcp_integrations`, sent as a header:

```
X-MCP-Token: <token>
```

SearchApi's own guidance: *"Do not put credentials in an MCP URL; URLs can be retained in
browser history and intermediary logs."* The `?token=` query form works but should not be used.

An unauthenticated call returns HTTP 401 with
`WWW-Authenticate: Bearer resource_metadata="…/.well-known/oauth-protected-resource/mcp", scope="mcp"`.

## 3. Scope the tool surface with a bundle

The server advertises 80+ tools. Do not attach all of them. In the dashboard, assemble a
**bundle** — the set of tools attached to one assistant — around the job at hand. A bundle is
SearchApi's substitute for fine-grained authorization: OAuth gives you one scope, the bundle
decides what that scope reaches.

Limit: **10 MCP integrations per account.**

## 4. Pick the tool

Every tool is the same underlying REST operation with the `engine` pinned. Choose by job:

| Job | Tool |
| --- | --- |
| General web grounding | `google_search` |
| Recency / news | `google_news` |
| Citable academic sources | `google_scholar`, `google_patents` |
| Places, hours, reviews | `google_maps`, `apple_maps` |
| Product price and availability | `google_shopping`, `amazon_search`, `walmart_search`, `ebay_search`, `bestbuy_search` |
| Travel | `google_flights`, `google_hotels`, `airbnb_search` |
| Labour market | `google_jobs` |
| Demand / interest over time | `google_trends` |
| Video | `youtube_search` |
| Second index for corroboration | `bing_search` |

The full binding of tool → REST operation is in `mcp/searchapi-tool-crosswalk.yml`.

## 5. Know the blind spot

**Every tool call counts as one search against the plan quota**, and there is no MCP tool for
the Account API. An agent driving this server cannot see its own remaining credits or how close
it is to the hourly cap (20% of monthly credits per hour) without stepping outside MCP and
calling `GET https://www.searchapi.io/api/v1/me` with the REST API key.

If your agent runs unattended, wire that check in separately. Nothing in the MCP surface will
warn it.

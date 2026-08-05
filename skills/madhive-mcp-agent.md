---
name: Drive Madhive through its MCP server
description: Connect an AI assistant to the hosted Madhive MCP server and use its 22 campaign-management tools safely, knowing which REST operations sit behind each one.
api: openapi/madhive-mcp-openapi-original.json
operations:
  - mcpJsonRpc
  - mcpSse
  - healthCheck
---

# Drive Madhive through its MCP server

Madhive runs a hosted Model Context Protocol server at `https://api2.madhive.com/mcp`.
It is the same campaign core as the REST API, projected as 22 tools over JSON-RPC 2.0.

## Connect

Onboarding is gated — client credentials are issued by a Madhive Account Manager.
Two OAuth flows are supported:

- **Client credentials** (machine-to-machine): backend services, automated workflows,
  batch processing. POST `client_id` + `client_secret` to
  `https://api2.madhive.com/oauth/token`, receive a JWT, send it as
  `Authorization: Bearer <token>` on every MCP request.
- **Authorization code with PKCE** (user-delegated): interactive developer tools.
  Start at `https://api2.madhive.com/oauth/authorize`, exchange the code plus the PKCE
  verifier for a JWT. A custom redirect URI must be registered with Madhive first.

Documented clients: Claude Code (`claude mcp add --transport http --client-id ...`),
Claude Desktop/Web (custom connector, Client ID under Advanced Settings), Gemini CLI
(`~/.gemini/settings.json` with `oauth.enabled` and `clientId`), Gemini Enterprise
(Custom MCP Server data store), and custom agents.

## Protocol

- `POST /` — JSON-RPC 2.0 (`mcpJsonRpc`). `{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}`
  to discover tools, `tools/call` to execute one.
- `GET /` — SSE streaming endpoint (`mcpSse`).
- `GET /health` — health check (`healthCheck`), also token-gated.

An unauthenticated `tools/list` returns `401 {"error":"unauthorized"}`, so the real
input schemas are only visible once authenticated. Call `tools/list` at session start
rather than assuming argument shapes.

## The tool surface

Read: `get_campaigns`, `get_campaign_details`, `get_advertisers`, `get_agencies`,
`get_creatives`, `get_audiences`, `get_lineitem_info`, `get_pixel`,
`get_publisher_groups`, `get_retargeting`, `list_publishers`, `list_segments`,
`list_orgs`, `list_metros`, `list_products`.

Write: `manage_campaign`, `manage_advertiser`, `manage_agency`, `manage_creative`,
`manage_audience`, `manage_publisher_group`, `manage_pixel`.

Each `manage_*` tool multiplexes create, update and archive. `manage_campaign` also
covers line items. Treat every `manage_*` call as a state-changing action: confirm the
verb and the target id with the user before invoking, because **archive is not
reversible through the API**.

`mcp/madhive-tool-crosswalk.yml` maps each tool to the REST `operationId`s behind it —
use it to reason about what a tool will actually do and what its real arguments are.

## What the tools do NOT cover

Present in REST, absent from the tool list — fall back to the REST API for these:

- Line item goal and schedule patches (`updateLineItemImpression`,
  `updateLineItemBudget`, `updateLineItemSchedule`)
- Creative-to-line-item scheduling and activation (`createCreativeLineItem`,
  `updateCreativeLineItem`, `activateCreativeById`, `deleteCreativeLineItem`)
- Creating or editing retargeting segments (read-only via `get_retargeting`)
- Optimization templates / Supply Guardrails
- Contextual segments and stations
- Media Trust creative scanning

## Operating rules

- Madhive publishes **no OAuth scopes**, so a token is not least-privilege — it carries
  whatever the client is entitled to. Do not hand an MCP token to an untrusted agent loop.
- There is **no idempotency contract**. If a `manage_*` call fails ambiguously, read the
  resource back with the matching `get_*` tool before retrying.
- Rate limits are per second by account tier (Basic 2 → Premium 15) and are enforced on
  the same gateway. Do not parallelize tool calls.
- Times are UTC; start dates cannot be in the past.
- Check `https://madhive.checkly-status-page.com/` when reads start failing with 5xx.

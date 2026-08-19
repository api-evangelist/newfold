---
name: newfold-mcp-gateway-session
description: Open an MCP session against a Newfold-powered WordPress site and discover
  what it can do, using the three-tool BLU gateway rather than a flat tool list.
generated: '2026-08-13'
method: generated
source: https://github.com/newfold-labs/wp-module-mcp/blob/main/README.md
api: newfold:blu-mcp
endpoint: https://{site}/wp-json/blu/mcp
operations:
  - initialize
  - notifications/initialized
  - tools/list
  - blu-list-abilities
  - blu-get-ability-schema
---

# Open a session and discover abilities

The Newfold BLU MCP server does not expose its ~83 abilities as tools. It exposes
**three gateway tools**, and everything else is reached through them. Getting this
wrong is the single most common failure: `blu-posts-search` is an *ability*, not a
tool name, and calling it at the `tools/call` level fails.

## 1. Establish the session

Sessions are mandatory and are not per-call.

```
POST /wp-json/blu/mcp
Authorization: Bearer <token>
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"initialize",
 "params":{"protocolVersion":"2025-06-18","capabilities":{},
           "clientInfo":{"name":"my-client","version":"1.0"}}}
```

The response carries an `Mcp-Session-Id` header. Send `notifications/initialized`
with that header, then put it on **every** later request. Sessions expire after
24 hours of inactivity, and a user may hold at most 32. On
`Invalid or expired session`, re-run both steps — do not retry the failing call.

`GET` on this route returns **405**; SSE is not implemented. Use POST.

## 2. Identify the gateway tools by SHAPE, not by name

Call `tools/list`. Three tools come back. Newfold explicitly tells clients not to
hardcode the default names — identify each role from its `inputSchema`:

| Role | Default name | Identify by |
|---|---|---|
| List | `blu-list-abilities` | optional `search` + `name_prefix`, no `ability_name` |
| Schema | `blu-get-ability-schema` | requires `ability_name`, no `parameters` |
| Call | `blu-call-ability` | requires `ability_name`, optional `parameters` |

## 3. Discover with a filter, always

Both discovery tools set `additionalProperties: false`, so send only documented
fields. Filters are AND-composed, and an unfiltered call is expensive:

```json
{"method":"tools/call","params":{"name":"blu-list-abilities",
 "arguments":{"name_prefix":"blu-media","search":"upload"}}}
```

- `name_prefix` — cheapest filter. `blu-posts`, `blu-media`, `blu-users`,
  `blu-global-styles`, `blu-wc-` (Bluehost WooCommerce wrappers),
  `woocommerce-` (WooCommerce-native abilities). Slash form is normalized.
- `search` — substring over name, label **and description**, so it finds
  abilities whose names say nothing about the task.

Read the result from `result.structuredContent.message` (parsed) or
`result.content[0].text` (the same thing as a JSON string).

## 4. Read the real input schema before calling

```json
{"method":"tools/call","params":{"name":"blu-get-ability-schema",
 "arguments":{"ability_name":"blu-posts-search"}}}
```

`message.input_schema` is the ability's actual JSON Schema, plus `annotations`
(watch for `readonly`). Never guess parameters — the schema is the contract.

## Errors

Two shapes, and they are easy to confuse:

- **Protocol** — a JSON-RPC `error` object, e.g. `-32602 Tool not found: foo`.
- **Execution** — a *successful* JSON-RPC response whose `result` carries
  `"isError": true`, e.g. `Access denied for tool: blu-call-ability`.

A client that only checks for `error` will read a permission denial as a success.

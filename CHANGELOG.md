# Changelog

Two version numbers move here:

- **Plugin** (`.claude-plugin/plugin.json`): the Claude plugin, its skill and this reference. Each version from 1.0.0 is tagged `vX.Y.Z`, with a GitHub release from 1.3.0.
- **MCP server** (`server.json`, and `serverInfo.version` from `https://stock-compass.app/api/mcp`): raised when the tools change, by their names, arguments or what they return. A new tool or field is a minor version; a removed or renamed one is a major version.

The REST API stays at `/api/v1`: fields and parameters are only added, so a client written for an earlier version keeps working. Dates are UTC.

## 1.3.0 · 2026-11-13 (MCP server 2.1.0)

- Three read tools: `get_events` (the change log), `get_account` (the cash in each market) and `get_dividends`. 20 tools: 11 that read and 9 that write.
- An argument a tool does not take is named in `warnings`, on the result and on a write's preview. REST names unused body fields (also inside a batch's trades) and unused query parameters the same way.
- `error.reason` tells apart the cases of one code, such as `exists`: `on_watchlist`, `external_id_recorded`, `external_id_in_batch`, `same_alert`.
- Rate limits: `RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset` on every answer; `Retry-After` gives the seconds left. Over MCP, a 429 is a JSON-RPC error with the request's id.
- The OpenAPI document names each write's scope, the rate-limit headers and `warnings`.

## 1.2.0 · 2026-11-12 (MCP server 2.0.0)

- The tools after the ChatGPT review freeze: 17 tools, with the journal, its stats, strategies, and changing and deleting price alerts.
- Protocol 2026-07-28 alongside the 2025 handshake versions; `server/discover` without a token.
- Every tool has an `outputSchema`; tool errors carry `structuredContent.error`.
- A write's preview runs every check that only reads, so most refusals come before the user is asked.
- API keys can be limited to some kinds of change: `trades:write`, `watchlist:write`, `positions:write`, `alerts:write`.
- Clients can identify themselves by a metadata document URL (CIMD).
- REST: `GET /events`, atomic batches (`"atomic": true`), the history of a holding's target and where each change came from.
- The skill is served over MCP (Skills extension), and a write's preview can show as a card with Confirm and Cancel (MCP Apps).

## 1.1.x · 2026-10-25 to 2026-10-31 (MCP server 1.0.0)

- The skill covers all of Stock Compass for any AI: strategies, backtests, fees, discipline, alerts, plans and the API.
- MCP writes take two calls, a preview and then the change with its `preview_id`.
- The REST reference: external ids, fees the broker charged, correcting and deleting trades, signal runs and the conditions that fired.
- The server is in the official MCP Registry as `io.github.Stock-Compass/stock-compass`.

## 1.0.0 · 2026-10-25

- The Stock Compass plugin for Claude: the MCP connector and the skill.

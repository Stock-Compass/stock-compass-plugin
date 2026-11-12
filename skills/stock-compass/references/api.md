# Connecting to Stock Compass: OAuth, API keys, MCP and REST

Read this when the user wants to connect an AI app, an agent or a script to Stock Compass, or asks about API keys, rate limits or the REST endpoints.

On the website: [connecting AI apps](https://stock-compass.app/en/ai) (which plans of each app can, and the steps by hand) · [the setup guide an AI follows](https://stock-compass.app/ai/setup.md) · [stock-compass.app](https://stock-compass.app).

**Labels.** Buttons are named as the English site shows them, followed by the Chinese site's label in brackets. When you point the user to one, use the label for the language their site is in.

All access is to the signed-in user's own records, and needs a Stock Compass **Pro** account (or an admin account). The plan is checked on every call, so access stops when Pro lapses.

## Two ways in

| | OAuth connection | Personal API key |
| --- | --- | --- |
| Best for | AI apps that support MCP with OAuth (Claude, ChatGPT, many others) | Scripts, command-line agents, and apps that can only send a fixed header |
| How it starts | The app opens stock-compass.app; the user signs in and approves | The user creates a key in Settings → AI apps & API (设置 → AI 应用与 API) |
| Secret the user handles | None | The key, shown once |
| Read only or read and write | Chosen on the approval page; read only by default | Chosen when the key is created, under **Access** (权限): read only, every change, or some kinds of change |
| Works with | `/api/mcp` | `/api/mcp` and `/api/v1` (REST) |
| Ends | Disconnect in Settings → AI apps & API | Revoke in the same place, or the key's expiry |

Both allow 60 requests a minute. The count is kept separately for each key and each connected app, not for the account: two keys of one user each get 60. Over MCP every message counts, `initialize` and `tools/list` included, and a write takes two calls (a preview, then the change), so recording 30 trades takes 60 or more. Space a long list of writes about a second apart. Every answer to a request with a key or token says where the count stands: `RateLimit-Limit` (60), `RateLimit-Remaining` (requests left in this minute) and `RateLimit-Reset` (seconds until the minute ends). Over the limit, the answer is HTTP 429 with `Retry-After`, the seconds until requests are taken again.

## Quick start (REST)

1. Create a key on the site: Settings → AI apps & API (设置 → AI 应用与 API), for example https://stock-compass.app/hk/settings/api-keys. Choose **Read and every change** (读取和所有修改) under **Access** (权限) to try a write.
2. Keep it out of your code:
   ```bash
   export STOCK_COMPASS_KEY=sc_…
   ```
3. Read:
   ```bash
   curl -s https://stock-compass.app/api/v1/me -H "Authorization: Bearer $STOCK_COMPASS_KEY"
   curl -s "https://stock-compass.app/api/v1/watchlist?market=hk" -H "Authorization: Bearer $STOCK_COMPASS_KEY"
   ```
4. Record a buy of a stock on the watchlist, with your own id so that a retry cannot record it twice. This is a real trade in the account: cash and the holding move.
   ```bash
   curl -s https://stock-compass.app/api/v1/trades -H "Authorization: Bearer $STOCK_COMPASS_KEY" \
     -H "Content-Type: application/json" \
     -d '{"market":"hk","ticker":"700","type":"buy","price":382.4,"quantity":100,"trigger":"other","external_id":"quickstart-1"}'
   ```
   Sent again, it records nothing and answers 409 `exists` with the trade already recorded.
5. Undo it, with the `trade_id` from step 4; cash and the holding go back:
   ```bash
   curl -s -X DELETE https://stock-compass.app/api/v1/trades/TRADE_ID -H "Authorization: Bearer $STOCK_COMPASS_KEY"
   ```

Errors come as `{ "error": { "code", "message" } }`; see [Errors](#errors).

Recording a trade through either one writes it into Stock Compass. Neither ever places an order with a broker.

## API keys

- A key looks like `sc_` followed by 43 letters, digits, `-` or `_`. It is shown once, when it is created. Stock Compass stores only a SHA-256 hash of it, so it cannot be shown again.
- When creating a key, the user gives it a name (up to 60 characters) and picks an expiry: 30 days, 90 days, 1 year, or never. **Access** (权限) sets what the key may change:
  - **Read only** (只读): nothing.
  - **Read and every change** (读取和所有修改): every write (scope `write`).
  - **Read and some changes** (读取和部分修改): only the kinds ticked: **Record trades** (记交易, `trades:write`: `POST`/`PATCH`/`DELETE /trades`, `POST /trades/batch`), **Change the watchlist** (改自选, `watchlist:write`: `POST /watchlist`, `DELETE /watchlist/{ticker}`), **Change stops and targets** (改止损和目标, `positions:write`: `PATCH /positions/{ticker}`) and **Change price alerts** (改价格提醒, `alerts:write`: `POST`/`PATCH`/`DELETE /alerts`). Cash and dividends need every change. A trade sent with `add_to_watchlist: true` for a stock not on the watchlist needs `watchlist:write` too. Every kind ticked is the same as every change.
  - The same scopes hold over MCP with that key. An app connected over OAuth is either read only or read and write.
- A user can have at most 5 active keys. Revoking a key takes effect at once.
- In the last 14 days before a key expires, every REST answer carries an `X-API-Key-Expires-At` header (an ISO time). `GET /me` gives the key's and the plan's end dates. A revoked or expired key gets 401 `key_revoked` or `key_expired`.
- A key with an end date that was used in the last 30 days also gets a reminder to the account's email address 7 days before it expires, and a last one in its final 24 hours (weekends not counted, so a key ending on a weekend gets it before the weekend). These are account emails: the user can turn them off in the app's email settings.
- Send the key as a header on every request: `Authorization: Bearer sc_…`.
- Treat a key as a password. Keep it in the agent's or the script's own secret settings, never in a conversation, a shared file or a repository. If it leaks, revoke it.

## MCP

The MCP endpoint is `https://stock-compass.app/api/mcp`. It uses Streamable HTTP, is stateless and answers in JSON.

- **With OAuth**: add the address as a remote MCP server and choose OAuth. The client finds the sign-in details itself. A request without a token gets HTTP 401, with a `WWW-Authenticate` header that points to `https://stock-compass.app/.well-known/oauth-protected-resource/api/mcp`. The one exception is `server/discover` (protocol 2026-07-28), which answers without a token: it gives only the versions, capabilities and instructions.
- **With an API key**: add the address as a remote MCP server and set the header `Authorization: Bearer sc_…`. For example, in Claude Code:
  ```bash
  claude mcp add --scope user --transport http stock-compass https://stock-compass.app/api/mcp --header "Authorization: Bearer sc_…"
  ```
  `--scope user` makes it available in every folder; without it, only in the one the command ran in. Other clients have their own setting for request headers.
- **For client developers**:
  - A client may use an https URL as its `client_id` instead of registering (Client ID Metadata Documents; the authorization server metadata has `client_id_metadata_document_supported: true`). The document there must give the same `client_id`, a `client_name` and one to ten `redirect_uris`, and no client secret (`token_endpoint_auth_method` `none`). It is fetched at each authorization request, without following redirects, within 5 seconds and 10 KB; if it cannot be fetched, the copy from the last day is used. The approval page names the site that serves the document. Claude Code and VS Code connect this way.
  - Clients register themselves (dynamic client registration). They are public clients, with no secret, and PKCE with `S256` is required. Redirect URIs (one to ten) are https, http on localhost, or an app scheme, with no fragment. Registrations from sites other than the known apps are limited to 10 an hour from one address.
  - An access token lasts an hour. A refresh token lasts 90 days and is replaced at each use; an old one sent again revokes that connection's tokens. So a connection unused for 90 days has to be approved again.
  - A request with no scope asks for both read and write; the user still chooses on the approval page.
  - MCP protocol versions: 2026-07-28, and the handshake versions 2025-11-25, 2025-06-18 and 2025-03-26, on the same endpoint. With 2026-07-28 there is no `initialize`: each request names the version in `params._meta` and in the `MCP-Protocol-Version` header, and sends `Mcp-Method` (and for `tools/call`, `Mcp-Name`), which must match the body (otherwise 400, `-32020`). `server/discover` gives the versions, capabilities and instructions; `tools/list` and `server/discover` carry `ttlMs` (10 minutes) and `cacheScope: "private"`. An unsupported version gets 400 with `-32022` and the supported list. The server's `version` (2.0.0 since November 2026) is in `initialize`, and with 2026-07-28 in each result's `_meta`.
  - `/api/v1` sends no CORS headers: an API key does not belong in a web page.
  - Resources: this skill is served over MCP too (Skills extension `io.modelcontextprotocol/skills`): `skills/list` and `skills/get` give `skill://stock-compass/SKILL.md` with a manifest of its files (SHA-256 and size), and `resources/read` serves each file, the same bytes as this repository's `skills/stock-compass`. `server/discover` declares the extension.
  - MCP Apps (`io.modelcontextprotocol/ui`): every write tool's `_meta.ui.resourceUri` is `ui://stock-compass/preview-card.html`. An app that renders MCP Apps shows a write's preview as a card with **Confirm** and **Cancel**; Confirm sends the same tool with the change and its `preview_id`, and puts the outcome in the model's context (`stockCompassPreviewCard`: `confirmed_and_made`, `refused` or `cancelled_by_user`). Apps without MCP Apps get the text answer as before.

### Connecting common apps

| App | How to connect |
| --- | --- |
| Claude (web, desktop app, Cowork) | Install the Stock Compass plugin: Customize → Plugins → Add → Add marketplace → `Stock-Compass/stock-compass-plugin`, then connect it on the plugin's Connectors tab. Or add the address directly as a custom connector. |
| Claude Code | `claude plugin marketplace add Stock-Compass/stock-compass-plugin`, then `claude plugin install stock-compass@stock-compass`; in Claude Code, `/reload-plugins`, then `/mcp` to sign in. Or `claude mcp add --scope user --transport http stock-compass` with the address. |
| ChatGPT | Start Pro or the free trial on stock-compass.app first: ChatGPT's approval page cannot start one. Then chatgpt.com/plugins → + → Add custom MCP server. Paste the address and choose OAuth. Which plans can add one is OpenAI's rule; as its [help page](https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt) put it on 2026-10-31: on the web, Business, Enterprise and Edu workspaces (on Business, admins and owners), and a personal Pro account for reading only. |
| Any other MCP client | Add the address as a remote MCP server (Streamable HTTP), with OAuth or with the API key header. |
| Scripts, or tools without MCP | Use the REST API below with an API key. |

These menu paths are the apps' own and can change. If one has moved, look for the app's setting for adding a remote MCP server or connector.

**The approval page.** The app opens stock-compass.app, where the user signs in, or signs up, and approves. Access is read only by default, and the button reads **Allow reading** (允许读取). To allow changes, the user ticks **Make changes** (修改) first, and the button becomes **Allow reading and changes** (允许读取和修改). **Deny** (拒绝) refuses. An account that is not on Pro but can still start the free trial sees **Start the N-day free trial and continue** (免费试用 N 天并继续). With ChatGPT it does not: by OpenAI's rules for apps in ChatGPT, the page only says the account is not on Pro and offers **Back to ChatGPT** (返回 ChatGPT). The user starts the trial on the site, then connects again.

### Tools and endpoints

The MCP tools map one to one onto the REST endpoints below:

| MCP tool | REST endpoint |
| --- | --- |
| `get_watchlist` | `GET /watchlist` |
| `get_signals` | `GET /signals` |
| `get_trades` | `GET /trades` |
| `get_price_alerts` | `GET /alerts` |
| `get_events` | `GET /events` |
| `get_account` | `GET /account` |
| `get_dividends` | `GET /dividends` |
| `add_to_watchlist` | `POST /watchlist` |
| `remove_from_watchlist` | `DELETE /watchlist/{ticker}` |
| `record_trade` | `POST /trades` |
| `edit_trade` | `PATCH /trades/{id}` |
| `delete_trade` | `DELETE /trades/{id}` |
| `set_stop_target` | `PATCH /positions/{ticker}` |
| `get_journal` | `GET /journal` |
| `get_journal_stats` | `GET /journal/stats` |
| `get_strategies` | `GET /strategies` |
| `get_strategy` | `GET /strategies/{id}` |
| `create_price_alert` | `POST /alerts` |
| `edit_price_alert` | `PATCH /alerts/{id}` |
| `delete_price_alert` | `DELETE /alerts/{id}` |

Setting cash and recording or deleting dividends (`PUT /cash/{market}`, `POST`/`DELETE /dividends`), `POST /trades/batch`, `POST /reconcile/positions` and `GET /settings/trading-costs` are REST only.

Tool arguments use the same names as the REST fields, and a tool returns what its endpoint returns. An argument the tool does not take is named in `warnings` on the result, and on a write's preview, so it can be put right before the user agrees. Each tool's `outputSchema` in `tools/list` describes its `structuredContent`: every field its result, a write's preview and a tool error can carry, none required at the top, so one schema covers all three. In MCP, `types` for `get_signals` is a list. Three reads page by default so one answer stays small: `get_watchlist` 100 stocks, `get_signals` 25 signals and `get_events` 25 changes (REST returns every match without `limit`, or 100 changes); `total` and `next_offset` say whether there is more.

A tool error comes back as a result marked as an error. Its text is the message described under "Errors" below, and `structuredContent.error` has the same `code` and `message`, with anything REST sends beside the error (such as `later_trades`) next to it. Where a REST message names an endpoint, the tool's names the tool that does it. Besides REST's codes: `insufficient_scope` (a read-only connection tried to write: connect the app again and allow changes on the approval page, or use a key with writes) and `plan_required` (the account is not on Pro).

**Writes over MCP take two calls.** Sent without `preview_id`, a write tool changes nothing. It answers with `status: "confirmation_needed"`, the `change` as it would be made (for `record_trade`, with the date it defaults to), for `edit_trade` and `delete_trade` the `trade` it changes, for `edit_price_alert` and `delete_price_alert` the `alert`, and a `preview_id` (`pv_` and 12 hex digits). Once the user agrees, the same arguments sent again with that `preview_id` make the change. A `preview_id` fits only that change on that connection, and lasts 10 to 20 minutes. Sent with any other change, or late, it changes nothing and comes back with a new preview. `record_trade` also returns a `note` that no order is placed. The preview first makes every check that only reads, so most refusals come on the first call, without a `preview_id`: not on the watchlist, a stock the quote source does not know, a sale of more shares than are held today, a stock still held, the same alert already set, a plan's alert, no open position, a trade out of order. What only the database can tell (a sale dated earlier against the shares held then, the plan's limits) is still checked on the second call. The REST API has no previews.

## REST API

Base URL: `https://stock-compass.app/api/v1`. It accepts **API keys only**; OAuth tokens work only with `/api/mcp`. An OpenAPI 3.1 description is at `GET /api/v1/openapi.json`, without a key; tools such as Postman import it. It names each endpoint's scope, the rate-limit headers and `warnings`; the answers' full shapes are here.

- Requests and responses are JSON.
- `GET` needs the read scope; `POST`, `PATCH` and `DELETE` need write.
- `market` is `hk`, `us` or `cn`.
- New fields and parameters are optional, so a client written for an earlier version keeps working.
- A write sent with a field it does not use still goes through, and its answer has `warnings`, such as `["Ignored field, not used by this endpoint or not for this request: trades[1].qty."]`, so a misspelt field is not lost silently. It names fields inside a batch's trades too, and fields a request had no use for, such as `trigger` on a sale. A trade the batch refused is left out, since it was not read to the end. With `dry_run` the warnings come back as well, so a batch can be checked before it is sent for real. Query parameters a request does not use are named the same way, in their own line of `warnings` (`?tickr=AAPL` would otherwise answer for every stock).
- Every list answers with `total`, the number of items that match before paging, and `next_offset`, the `offset` of the next page, or null on the last one.

### Tickers

| Market | Send | Stored and returned as |
| --- | --- | --- |
| `hk` | 1–5 digits, with or without `.HK`: `700`, `0700.HK` | `0700.HK` |
| `cn` | 6 digits, with or without the suffix: `600519` | `600519.SS` (6xxxxx, Shanghai), `000001.SZ` (0xxxxx, 3xxxxx, Shenzhen) |
| `us` | 1–5 letters; a share class with a dot or a dash: `BRK.B` | `BRK-B` (dots become dashes) |

In a path (`DELETE /watchlist/{ticker}`, `PATCH /positions/{ticker}`) the market is read from the ticker: `.HK` is Hong Kong, `.SS` or `.SZ` is China, anything else is the US. So `PATCH /positions/ANF` works as it is; a Hong Kong or China stock needs its suffix (`0700.HK`), or `market` in the query or body.

### Dates and times

- Dates are `YYYY-MM-DD`, in the market's own time zone: Asia/Hong_Kong, Asia/Shanghai, America/New_York.
- `traded_at` is stored as sent. Stock Compass does not move a pre-market, after-hours or overnight fill to another day, so send the trade date the broker reports for the fill. Without `traded_at`, it is today in the market's time zone.
- A trade has a date, and a time when the client sends `executed_at` with it (`POST /trades`). Trades on one date count in the order they happened: by `executed_at`, and a trade without one at the time it was recorded, since it was made no later than that. That order decides the holding on the day, whether a sell is covered, and which journal entries a sale closes. Without times, record a day's trades in the order they happened. A trade that would come before the stock's later trades gives 409 `out_of_order` (see `POST /trades`).
- `recorded_at` and `created_at` are UTC timestamps.

### Cost, fees and P&L

- **Fees**: send `fees` with what the broker charged for a trade, and it is kept on the trade (`fees_source: "provided"`). Without it, the fee is worked out from the user's trading-cost settings for the market (`"estimated"`) and kept on the trade as it was then: a later change to the settings does not change it. Either way it moves cash, and `POST /trades` returns it. Trades recorded on the website are estimated the same way.
- **A holding's `cost_price`** is the weighted average of its buy prices, without fees. A sale does not change it; after a sale to zero, the next buy starts a new average.
- **P&L on the website** is net of fees. A holding's buy fees are the fees kept on its buys, reported or estimated when recorded, still in it (a sale takes its share, as it does of the cost); only shares from trades with no fee kept are estimated from the current settings. The sell fee of a holding is always an estimate.
- **The journal** keeps each buy as its own entry, at its own price. A partial sale closes the oldest open entries first (FIFO), so the return of each closed entry uses that entry's price. A broker that reports realized P&L on average cost can show a different figure for a partial sale.
- **The journal's net figures** (the cumulative return and the realized P&L curve) use the fees kept on an entry's buy and sale, in proportion to its shares, when its trades can be told apart: the buy it was logged for, or the one buy on its date at its price; the one sale on its close date at its price. One buy sold in three parts is charged that one order's fee across the three entries. Otherwise they estimate from the current settings. The win rate and the rules-vs-gut comparison use price returns without fees.

```bash
curl -s https://stock-compass.app/api/v1/me -H "Authorization: Bearer $STOCK_COMPASS_KEY"
```

### GET /me

What the key can do, and when the key and the plan end (`null`: never).

```json
{ "user_id": "…", "scopes": ["read", "write"], "requests_per_minute": 60,
  "key": { "id": "…", "name": "Broker sync", "expires_at": "2027-01-20T00:00:00Z" },
  "plan": { "tier": "pro", "expires_at": null } }
```

### GET /watchlist

Query, all optional:
- `market`;
- `holding`: `true` for holdings only, `false` for stocks not held;
- `limit` (1–1000) and `offset`: one page. Without `limit`, the whole list.

`total` is the number of stocks that match, before paging; `next_offset` is null on the last page, and always without `limit`. `limit` and `offset` come back when a page was asked for.

`stop` and `target` are the ones in force on the open position; they are null when the stock is not held. A holding also has:
- `opened_at`: the date of the oldest open journal entry, when the position was opened;
- `journal_entry_id`, `entry_stop`, `entry_target`: the latest buy's journal entry, whose levels are in force, with the stop and target set when it was bought;
- `stop_history` and `target_history`: every change of that entry's stop, and of its target, oldest first, from any source. `from` or `to` is null when there was none. Each change has `source`: `web` (the website), `api` (the REST API) or `mcp` (an app over MCP); null for changes from before 12 November 2026, and target changes are kept only from then. A confirmed split rescales the levels without adding a change;
- `exit_mode`: how the holding's exits are watched, `strategy`, `levels_only` or `external` (see `PATCH /positions/{ticker}`);
- `target_reached_at` and `stop_breached_at`: when the price reached the target, or went through the stop, in force, from the alert the website keeps on it (`from_plan` in `GET /alerts`); null while it has not, and while there is no target or stop. A new level, or re-arming its alert, starts over. With `exit_mode: external` the levels have no alerts, so both stay null.

```json
{ "items": [
  { "ticker": "0700.HK", "market": "hk", "name": "TENCENT", "name_zh": "腾讯控股",
    "holding": true, "quantity": 200, "cost_price": 382.4, "stop": 365, "target": 450,
    "opened_at": "2026-10-02", "journal_entry_id": 42, "entry_stop": 350, "entry_target": 450,
    "stop_history": [ { "at": "2026-10-15T08:10:00Z", "from": 350, "to": 358, "source": null },
                      { "at": "2026-11-13T08:05:00Z", "from": 358, "to": 365, "source": "mcp" } ],
    "target_history": [],
    "exit_mode": "strategy", "target_reached_at": null, "stop_breached_at": null,
    "added_at": "2026-09-01T02:00:00Z" }
], "total": 1 }
```

### POST /watchlist

Body: `{ "market": "hk", "ticker": "700" }`.

- The ticker must exist at its market's quote source, or the answer is 404 `unknown_ticker`.
- The plan's watchlist limit applies (403 `limit`).
- A stock already on the watchlist gives 409 `exists`.
- Answers 201 with `{ ticker, market, name, name_zh }`.

### DELETE /watchlist/{ticker}

Query: `market` (optional when the ticker carries its suffix, as in `0700.HK`).

- Refused while the stock is held (409 `holding`): record the sale first.
- Also deletes the stock's price alerts.
- Answers `{ ticker, market, removed: true }`.

### GET /signals

Query, all optional:
- `market`;
- `days`: 0–90, default 7;
- `from` and `to`: dates, to move or narrow the window. `from` defaults to `days` before `to`, and `to` to today; at most 90 days apart;
- `ticker`: one stock, in any form GET /trades takes (`700` is 0700.HK without `market`);
- `types`: comma-separated, any of `buy,add,exit,trim,price_alert`; default `buy,add,exit,trim`;
- `state`: `all` (the default), `new` or `active`. See runs below;
- `recorded_after`: a time, such as `2026-10-31T01:00:00Z`: only signals whose `recorded_at` is later. For polling, send the latest `recorded_at` you have; a buy for an earlier `date` can still arrive, since buys are recorded after the bar is final;
- `limit` (1–500) and `offset` (default 0): one page. Without `limit`, every match. `total` counts every match; `next_offset` is null on the last page.

Each market's buy, add, exit and trim signals come from the strategy that was active in that market on each date. Price-alert signals are the user's own alerts that fired. Results are newest first.

- `date` is the daily bar the signal was computed on, in the market's time zone. `recorded_at` is when the scan wrote it (UTC).
- Buy and add wait for that bar to be final: they are recorded by the scan after the close, the same evening (about 16:20 Hong Kong time for HK and A-shares, about 16:10 New York time for the US), or failing that by the next trading day's 09:00 scan. Exit and trim are recorded during the session, at the price when the scan ran.
- `days` counts back from the market's own date (for all markets, the earliest of them), not from UTC's.
- `strategy` and `strategy_id` name the strategy that fired; `GET /strategies/{strategy_id}` gives its rules.
- Why it fired: a strategy signal has `indicators`, a fixed set of indicator values when it was recorded, and `matched_conditions`, the rule's conditions that held. Each has `indicator`, `operator`, `threshold` (a number, a text, or the list of `IN` / `NOT IN`) and the `value` at the time (`null` if it was not a finite number); an old indicator name reads as its current one. They are the conditions that made the rule fire: every condition of an AND group, the first that held of an OR group, and with groups joined by OR, those of the first group that held. `matched_conditions` is `null` on signals recorded before it was added (November 2026). A price-alert signal has `alert`: the alert's `id`, `threshold`, `direction` and `note`, `plan` (`stop` or `target` when the alert is the plan's stop or target from the journal, otherwise `null`, also when the alert has been deleted since), and `touched` (the bar's low or high that crossed the level, and its date, when it was a bar and not the price at the scan).
- The answer gives the window's `from`, and `to` when one was sent.

**Runs.** A rule that stays true is recorded again on every bar: an exit rule that holds for a week gives five exit signals. Signals of one strategy, ticker and type on consecutive scanned bars form one run.
- `active_since` is the date of the run's first bar, and `occurrences` how many bars of the run there have been up to this signal (1 on the first).
- A run ends on a bar the scan evaluated without recording it. A day with no scan, such as an exchange holiday, does not end it. When the account has no scan records for the market in the period, weekdays are used instead, and a holiday then ends a run.
- Runs are traced back up to 30 days before `from`.
- `state=new` returns only the first bar of each run, `state=active` only the later ones. A price-alert signal is always a run of one.

```json
{ "from": "2026-10-18", "total": 2, "next_offset": null,
  "note": "Buy and add signals are recorded once their daily bar is final …",
  "signals": [
    { "id": 123, "ticker": "ANF", "market": "us", "type": "exit", "date": "2026-10-23",
      "price": 97.2, "strategy": "Trend", "strategy_id": "…", "recorded_at": "2026-10-23T17:05:00Z",
      "active_since": "2026-10-19", "occurrences": 5,
      "indicators": { "rsi_14": 71.2, "today_change_pct": 3.4, "price_vs_ma20": 8.1 },
      "matched_conditions": [ { "indicator": "rsi", "operator": ">", "threshold": 70, "value": 71.2 } ] },
    { "id": 124, "ticker": "AAPL", "market": "us", "type": "price_alert", "date": "2026-10-23",
      "price": 226.4, "strategy": null, "strategy_id": null, "recorded_at": "2026-10-23T15:05:00Z",
      "active_since": "2026-10-23", "occurrences": 1,
      "alert": { "id": "…", "threshold": 227, "direction": "below", "note": null, "plan": "stop",
                 "touched": { "price": 225.9, "date": "2026-10-23" } } }
  ] }
```

### GET /trades

Query: `market`, `ticker`, `from`, `to`, `external_id`, `tag` (all optional); `limit` (1–500, default 100); `offset` (default 0). `tag` keeps the trades that carry that label (see `tags` under `POST /trades`). A `ticker` without `market` finds its market from the code: `700` is 0700.HK, `600519` is 600519.SS, `BRK.B` is BRK-B.

Results are newest first: by date, and within a date by when they were recorded. `executed_at` is the time the trade was executed when the client sent one, otherwise null; it is UTC, written `…+00:00` here and `…Z` in the answer to `POST /trades`. `total` is the number of trades that match; `next_offset` is null on the last page. `external_id` is null for a trade recorded without one, such as on the website. `fees` is the fee the client reported, with `fees_source: "provided"`; it is null with `"estimated"` when none was. `estimated_fees` is then the estimate from the user's trading-cost settings that the trade was recorded with, as the website's P&L counts it (for an old trade with none kept, from the current settings); it is null when a fee was reported. `tags` is there only on a trade recorded with some.

```json
{ "trades": [
    { "id": "…", "ticker": "0700.HK", "market": "hk", "type": "buy", "price": 382.4,
      "quantity": 200, "traded_at": "2026-10-23", "executed_at": "2026-10-23T01:42:10+00:00", "recorded_at": "2026-10-23T03:00:00Z",
      "external_id": "FUTU-8231", "fees": 3.86, "fees_source": "provided", "estimated_fees": null }
  ], "limit": 100, "offset": 0, "total": 1, "next_offset": null }
```

### GET /strategies

The user's own strategies, as the website's Strategies page lists them. Query: `market` (optional).

- `active` is the strategy in use in its market now. `GET /signals` returns each date's signals from the strategy that was in use on that date, so older signals can come from another one.
- `rules_changed_at` is when its rules last changed.
- Answers `{ "strategies": [ { "id", "name", "description", "market", "active", "rules_changed_at", "created_at", "updated_at" } ], "total", "next_offset": null }`.

### GET /strategies/{id}

One strategy with its rules as they are now, so a signal's `strategy_id` leads to its strategy. A deleted strategy is still found, with `deleted_at` set, since its old signals name it. A signal recorded before `rules_changed_at` may have fired on earlier rules; its `matched_conditions` are what held then.

- `rules` has the four sections, `buy`, `add`, `trim` and `exit`, as the signal engine reads them: `outer_op` joins the `groups` (`OR`: any group; `AND`: all), and each group's `inner_op` joins its `conditions`. Each condition has `indicator` (an old indicator name reads as today's, as in `matched_conditions`), `operator`, `value` and `text`, its words in English and Chinese as the page's **View** shows them. A section without conditions has no groups; a group without conditions never holds, so in an `AND` section it keeps the whole section from firing.
- `export` is in the shape of the page's **Export JSON** (`name`, `description`, `market`, `rules`), with the same rules written out in full (every section as `{ "v": 2, "outerOp", "groups" }`, numbers as numbers, today's indicator names and sector codes), so the page's **Import** takes it as it is. A strategy with no buy conditions will not import: the page requires them.
- Another user's strategy gives 404 `strategy_not_found`. Creating and changing strategies through the API is not offered.

```json
{ "strategy": { "id": "…", "name": "Trend", "market": "us", "active": true, "deleted_at": null, … },
  "rules": {
    "buy": { "outer_op": "OR", "groups": [ { "inner_op": "AND", "conditions": [
      { "indicator": "rsi", "operator": "<", "value": 30, "text": { "en": "RSI(14) < 30", "zh": "RSI(14) < 30" } } ] } ] },
    "add": { "outer_op": "OR", "groups": [] }, "trim": { … }, "exit": { … } },
  "export": { "name": "Trend", "description": "…", "market": "us", "rules": { … } } }
```

### GET /journal

The trade journal, as the website's journal page lists it: one entry per position opened, newest opened first. A partial sale closes part of an entry as an entry of its own.

Query (all optional): `market`; `status` (`all`, the default, `open` or `closed`); `ticker`; `from` and `to`, dates the entry was opened; `trigger` (`signal`, `price_alert`, `manual`, `imported` or `other`); `tag`, entries with that label; `limit` (1–500, default 100) and `offset` (default 0).

`total` counts entries, as the journal page's list shows them: a buy sold in three parts is four. The page's own trade counts group those into one.

- `market` is the code's own market (`.HK`, `.SS`/`.SZ`, none for US), so an entry still counts after its stock left the watchlist.
- `trigger.type` is why the stock was bought: `signal` on a strategy signal (with its `signal_id`, `strategy_id` and `strategy` name), `price_alert` on one of the user's price alerts, `manual` the user's own decision, `imported` a holding from before the journal, logged from the journal page's banner, `other` none of these. The journal's **By rules** (按规则操作) side is `signal` only, **By gut feel** (凭直觉操作) `manual` only.
- `plan` is the stop and target written when the position was opened; `levels` are the ones in force now (null when cleared since); `stop_history` and `target_history` every stop and target change, oldest first, with its `source` (see `GET /watchlist`). The closed part of a partial sale is a new entry that keeps only the plan: its `levels` are the plan's, its histories are empty and its `trade_id` null.
- `realized_return_pct` is the closed entry's price return, without fees, as the journal page shows it; null while open. `close_reason`, `deviated_from_plan`, `deviation_reason` and `lesson` come from the website's **Close position** (平仓) form; a sale through `POST /trades` sets `close_reason` and `lesson` only.
- `tags` are the labels of the buy the entry was logged for (`[]` when none); the closed part of a partial sale keeps them.
- `trade_id` is the buy the entry was logged for, when known (`GET /trades`). `source` is `sample` for invented entries that an earlier **load sample data** button wrote into some older accounts (the execution record leaves them out), otherwise usually null.

```json
{ "entries": [
    { "id": 812, "ticker": "0700.HK", "market": "hk", "status": "closed",
      "opened_at": "2026-10-01", "open_price": 380, "shares": 100,
      "closed_at": "2026-10-20", "close_price": 399, "realized_return_pct": 5,
      "trigger": { "type": "signal", "signal_id": 7, "strategy_id": "…", "strategy": "Trend" },
      "thesis": "…", "plan": { "stop": 350, "target": 450 }, "levels": { "stop": 360, "target": 450 },
      "stop_history": [ { "at": "2026-10-06T01:00:00Z", "from": 350, "to": 360 } ],
      "close_reason": "rule_exit", "deviated_from_plan": false, "deviation_reason": null,
      "lesson": "…", "trade_id": "…", "source": null }
  ], "limit": 100, "offset": 0, "total": 1, "next_offset": null }
```

### GET /journal/stats

The journal page's figures for one market, worked out by the same code as the page and the Monday review email, so the three agree. Query: `market` (required), by the code's own market as the Monday email counts it; `from` (optional), entries opened on or after that date, as the page's periods; `group_by=tag` (optional) adds `by_tag`.

- `trades` and `closed` count decisions, as the page does: a buy sold in three parts is one trade.
- `cumulative_return_pct` is the closed entries' gain less their fees over what they cost, as the page's figure and equity curve: the fees kept with the trades, else the user's trading-cost settings (the market's default schedule when none are set). `avg_hold_trading_days` counts weekdays from open to close. Both are null with nothing closed.
- `by_rules`, `by_gut` and `by_price_alert` are the closed trades by why they were bought (`signal`, `manual`, and a signal that was a price alert; `imported` and `other` are in neither): `count`, `wins`, `win_rate_pct` and `avg_return_pct`, price returns without fees, as the page. With no trades on a side the two rates are null.
- `comparison.result` is the page's and the email's reading of the two averages: `insufficient` (with `needed_per_side`, each side needs that many closed trades), `close` (within 1 point), `rules_better` or `gut_better`, with `gap_pp`, the size of the difference in points (never negative: `result` says which side is ahead). It states the record; it is not advice.
- `by_tag`, with `group_by=tag`: the closed trades under each label, with `count`, `wins`, `win_rate_pct` and `avg_return_pct` as above, most trades first. A trade with two labels counts under both; the unlabelled ones are under `tag: null`.

```json
{ "market": "hk", "from": null, "trades": 24, "closed": 18,
  "cumulative_return_pct": 6.42, "avg_hold_trading_days": 11.5,
  "by_rules": { "count": 9, "wins": 6, "win_rate_pct": 66.67, "avg_return_pct": 3.1 },
  "by_gut": { "count": 7, "wins": 3, "win_rate_pct": 42.86, "avg_return_pct": -0.8 },
  "by_price_alert": { "count": 2, "wins": 1, "win_rate_pct": 50, "avg_return_pct": 1.2 },
  "comparison": { "result": "rules_better", "gap_pp": 3.9 } }
```

### POST /trades

Records a buy or sell the user made, exactly as the trade form on the website does. Cash, the holding, the lots and the trade journal all move together, in one transaction.

| Field | Required | Notes |
| --- | --- | --- |
| `market`, `ticker` | yes | The stock must be on the watchlist (409 `not_on_watchlist`), unless `add_to_watchlist` is true. |
| `type` | yes | `buy` or `sell`. |
| `price` | yes | Per share, without fees. |
| `quantity` | yes | Whole shares. |
| `traded_at` | no | The trade date in that market. Defaults to the date of `executed_at` in the market, else today there; cannot be in the future. |
| `executed_at` | no | When the trade was executed: a time with its offset, such as `2026-11-06T09:31:05+08:00` or `…Z`. It must fall on `traded_at` in the market's time zone and not in the future. It orders the trade within its date. See below. |
| `trigger` | buy only | `signal`, `manual` or `other`. See below. |
| `stop`, `target` | buy only | The planned stop and target. |
| `thesis` | buy only | Up to 1000 characters. |
| `close_reason` | sell only | `rule_exit`, `rule_trim`, `manual_stop`, `manual_target` or `other`. |
| `lesson` | sell only | Up to 1000 characters. |
| `add_to_watchlist` | no | `true`: for a buy of a stock not on the watchlist, add it first, as `POST /watchlist` does. Default `false`. |
| `external_id` | no | The client's own id for the trade, such as the broker's order number: 1 to 100 characters. See below. |
| `fees` | no | What the broker charged for this trade, zero or more, in its currency. Without it, estimated from the user's settings. |
| `exit_mode` | buy only | `strategy`, `levels_only` or `external`: how the holding's exits are watched from now on. See `PATCH /positions/{ticker}`. |
| `allow_out_of_order` | no | `true`: record a trade dated before trades already recorded for the stock, even though it changes them. See below. Default `false`. |
| `tags` | buy only | Up to 10 labels, each 1 to 40 characters without commas, such as the system or the kind of signal behind the trade. Kept on the trade and its journal entry; filter with `tag` and count with `GET /journal/stats?group_by=tag`. |

- **Fees**: `fees` is what the broker charged, zero or more, in the trade's currency; it moves cash and is kept. Without it, the fee is worked out from the user's own trading-cost settings for that market. The response returns the fee and `fees_source`. See [Cost, fees and P&L](#cost-fees-and-pl).
- **`trigger`** says why the user bought. The journal's **By rules** (按规则操作) vs **By gut feel** (凭直觉操作) comparison is built on it.
  - `signal`: on a strategy signal. Counts as by rules.
  - `manual`: the user's own decision. Counts as by gut feel.
  - `other`: neither, such as a rebalance. It counts on neither side.
  - Without `trigger`, the buy is filed as `signal` when the strategy in use in that market on the signal's date gave the stock a buy signal in the last 14 days, counted back from today, not from `traded_at` (for a stock already held, an add or buy signal; the strategy active now first), and otherwise as `manual`. Sending `signal` when there is none gives 422 `no_recent_signal`.
  - Holdings bought before the user connected are best sent with `other`: left out, they count as gut feel, or under a recent signal they did not follow.
- **`close_reason`** is the reason on the journal entry, as the website's **Close position** (平仓) form gives it. It is kept on the entry; the journal's statistics do not use it. **Exits against the plan** (按计划复盘退出) judges each exit by its price against the stop and target.
  - `rule_exit`: **Exit rule fired** (触发清仓规则), sold because the strategy's exit rule fired;
  - `rule_trim`: **Trim rule fired** (触发减仓规则), sold part on the trim rule;
  - `manual_stop`: **Manual stop-loss** (主动止损), sold at the user's own stop;
  - `manual_target`: **Manual take-profit** (主动止盈), sold at the user's own target;
  - `other`: **Other** (其他).
- **Selling** closes the oldest open journal entries first (FIFO) with `close_reason`, or with `other`, and puts `lesson` on them; selling to zero closes them all. If the answer lists `close_reason` and `lesson` under `ignored`, with a `journal_note`, they were not kept.
- **Selling more** than was held on `traded_at` gives 409 `oversell`. A sale of a stock not on the watchlist gives 409 `not_on_watchlist`, or with `add_to_watchlist: true`, 409 `oversell`: nothing is held.
- **With `add_to_watchlist`**, the stock is added only once the rest of the request is valid. The answer then has `"added_to_watchlist": true`; a stock the quote source does not know gives 404 `unknown_ticker`, and a full watchlist 403 `limit`.
- **`exit_mode`** is set on the holding once the buy is recorded, and the answer repeats it. If it could not be set, the buy still stands: the answer lists `exit_mode` under `ignored`, with `exit_mode_error`; set it with `PATCH /positions/{ticker}`. An unknown value, or `exit_mode` on a sale, gives 400 and records nothing; so does 503 `not_available` while exit modes are not available yet.
- **`external_id`** guards against recording a trade twice. It is unique per user and market.
  - Sent again, the same id records nothing and answers 409 `exists`, with the trade already recorded under it beside the error: `{ "error": { … }, "trade": { "id": "…", … } }`. That is so whatever else the request says.
  - So a request that timed out can be sent again as it was. Two requests at once with the same id record one trade.
  - Once that trade is deleted, the id is free again.
  - Without `external_id`, sending the same trade twice records it twice.
- **Same-day order**: trades on one date count by `executed_at`, and a trade without one at the time it was recorded. A trade entered on the website has no time, so it counts when it was entered: a client that sends times should leave the website's trades of that day alone, or a timed sale may count before a buy entered later on the website. See [Dates and times](#dates-and-times).
- **Out of order**: trades count in date order, so a trade dated before trades already recorded for the stock changes what was worked out for them. When it would, the request gives 409 `out_of_order` and records nothing: a buy dated before a later sale, or a sale dated before any later trade. A buy before later buys only is recorded, since buys add up the same in any order. The error lists the later trades in `later_trades` (up to 20, oldest first; `more_later_trades` is true when there are more). Send trades oldest first; to record one anyway, such as a missed trade, send it again with `allow_out_of_order: true`. Within one date it is checked when the trade has `executed_at`: a trade with an earlier time than the date's trades already recorded counts before them, so a buy at 09:30 sent after a sale at 10:00 gives `out_of_order`. A trade without a time counts after the date's trades already recorded, and is not checked within its date. Two requests for the same stock sent at once are not checked against each other: send a stock's trades one after another. MCP's `record_trade` checks it the same way. Moving a trade's date with `PATCH /trades/{id}` is not checked either, and drops its `executed_at`, which belonged to the old date.
- **`executed_at`** is kept with the trade and returned by `GET /trades` and in this answer. Before it is available a request with it gives 503 `not_available` and records nothing; without it the request works as before.

Answers 201:

```json
{ "trade_id": "…", "ticker": "0700.HK", "market": "hk", "type": "buy", "price": 382.4, "quantity": 200,
  "traded_at": "2026-10-23", "external_id": "FUTU-8231", "fees": 3.86, "fees_source": "provided", "shares_before": 0, "shares_after": 200, "cash": 12345.6,
  "journal_entry_id": 42, "trigger": { "type": "signal", "signal_id": 123, "signal_date": "2026-10-22" } }
```

For a sale, `journal_closed_shares` takes the place of `journal_entry_id` and `trigger`; `ignored` and `journal_note` appear only when `close_reason` and `lesson` were not kept. `external_id` is there only when one was sent.

### POST /trades/batch

Up to 50 trades in one request, which counts as one against the rate limit. Body: `{ "trades": [ … ], "on_error": "stop", "dry_run": false, "atomic": false }`; each trade is what `POST /trades` takes.

- The trades are recorded one at a time, in the order given, each exactly as `POST /trades` records it. One that is refused does not undo those before it, unless `atomic` is true.
- `on_error`: `stop` (the default) leaves the rest unattempted after a refusal, since a later trade of the same stock usually depends on it; `continue` tries them all.
- Give every trade an `external_id`: a batch sent again after a timeout then records nothing twice.
- Answers 200 with `{ dry_run: false, total, recorded, failed, not_attempted, stopped_at, results }`. Each result has `index`; `ok: true` with the `trade` as `POST /trades` answers it, or `ok: false` with the `status` and `error` it would have given.
- **`atomic: true`** records every trade or none. Each trade is checked as `POST /trades` checks it, and against the batch's earlier trades: the same `external_id` twice gives 409 `exists`, and a trade dated before an earlier trade of the same stock in the batch that it would change gives 409 `out_of_order` (send the batch in date order, or that trade with `allow_out_of_order: true`). Then all of them are recorded in one database transaction, with any stock added (`add_to_watchlist`), and the tags and exit modes saved in it too. A refusal there (an oversell by date, a full watchlist, an `external_id` already recorded) leaves nothing recorded. The answer has `atomic: true`: every trade recorded, or `recorded: 0` with the one trade refused in `results` and `stopped_at` its index. `on_error` does not go with it (400). A batch sent again after a timeout either was recorded whole (the first trade with an `external_id` then comes back 409 `exists`) or not at all.
- **`dry_run: true`** records nothing and says what each trade would be: `new` (no trade with its `external_id` yet; without one, this is all it can say), `match` (a trade with that `external_id` is recorded, with the same side, price, quantity and date), `conflict` (one is, and it differs; the `trade` is there), or `invalid` (it would be refused; `error` says why). It checks the fields, the `external_id` and the watchlist, not the holdings, so an oversell shows only when the batch is really sent. The answer counts each kind: `{ dry_run: true, total, new, match, conflict, invalid, results }`.

### POST /reconcile/positions

Compares the holdings recorded here with the ones a broker shows, for one market, and names the stocks where they differ. It changes nothing, so a read-only key may call it. Body: `{ "market": "hk", "positions": [ { "ticker": "700", "quantity": 200 } ] }`, up to 2000 positions; the same stock twice adds up (two accounts at one broker).

- Each difference has `ticker`, `kind`, `here` and `broker` (the shares on each side, null where it is not held) and `difference` (broker minus here). `kind` is `quantity` (both hold it, in different amounts), `missing_at_broker` (held here only: a sale not recorded?) or `missing_here` (at the broker only: a buy not recorded?).
- Costs are not compared: brokers average them differently.

```json
{ "market": "hk", "held_here": 3, "held_at_broker": 3, "matched": 1,
  "differences": [ { "ticker": "0005.HK", "kind": "quantity", "here": 400, "broker": 350, "difference": -50 } ] }
```

### GET /settings/trading-costs

Query: `market` (required). The user's trading-cost settings for the market, the ones `estimated_fees` is worked out from, or the market's default when none are saved, with the `formula` in words, so a client can work the same estimate out itself.

```json
{ "market": "hk", "currency": "HKD",
  "costs": { "broker": "moomoo", "commission_pct": 0.03, "per_share_commission": null, "min_commission": 3,
             "platform_fee": 15, "per_share_platform_fee": null, "min_platform_fee": null,
             "other_fees_pct": 0.018, "stamp_duty_pct": 0.1, "stamp_duty_on": "buy_and_sell" },
  "formula": "One side of a trade, value = price x quantity. …" }
```

### PATCH /trades/{id}

Corrects a recorded trade, as the trade list on the website does. Body: any of `price`, `quantity`, `traded_at` and `fees`; what is left out stays as it was.

- **Cash** moves by the difference.
- **Fees**: with `fees`, that is what the broker charged, and it is kept; `fees` alone is a correction too. Without it, a fee reported before is kept while the price and quantity stay the same (a date correction); otherwise the fee is worked out again from the user's current settings. A price or quantity correction on the website also goes back to the estimate.
- **The holding** is recomputed from all the trades.
- **The journal**: a buy's entry takes the new shares, and the new price and date where the entry still had the trade's old ones; a sale with the same quantity updates the entries it closed, with their returns. An entry that no longer matched the trade is left as it was, and `journal_review` says so.
- **The side and the ticker** cannot change (400): that is another trade. Delete it and record the right one.
- **A change that would leave a sell of shares not held at the time** is refused with 409 `oversell`, and nothing changes.
- `external_id` stays with the trade.

Answers 200:

```json
{ "trade": { "id": "…", "ticker": "0700.HK", "market": "hk", "type": "buy", "price": 381, "quantity": 200,
             "traded_at": "2026-10-23", "recorded_at": "…", "external_id": "FUTU-8231",
             "fees": 3.86, "fees_source": "provided" },
  "fees": 3.86, "fees_source": "provided", "cash": 12345.6, "shares_after": 200, "journal_updated": 1, "journal_review": null }
```

Nothing to change gives `{ "trade": {…}, "unchanged": true }`.

**`journal_review`** is null, or says which journal entries were left as they were for the user to check: `{ "kind": …, "date": "2026-10-23", "message": "…" }`.
- `kind` is a stable code; `message` is in English. Translate from `kind` for a user in another language.
  - `buy_entry`: the buy's journal entry is closed, or no longer matches the trade (it was edited, or part or all of it sold), so it was not changed.
  - `sell_close`: the journal entries this sale closed were not changed.
- `date` is the trade's date, where to look on the Journal page.
- A correction of `fees` alone never gives one: the journal keeps no fees, so nothing in it has to follow.

### DELETE /trades/{id}

Deletes a trade recorded by mistake, as the trade list on the website does.

- **Cash** gets back what the trade moved, and **the holding** is recomputed: deleting a buy puts the shares and cost back to what they were without it.
- **A buy's journal entry** is deleted with it while it is still exactly that buy (wholly open, same shares). Otherwise it stays, and `journal_review` says so.
- **A deleted sale does not reopen** the journal entries it closed: their close reasons and lessons are the user's. `journal_review` points at them.
- **A buy whose shares a later sale used** cannot be deleted: 409 `oversell`, and nothing changes. Correct or delete that sale first.
- The trade's `external_id` becomes free to use again.

Answers 200:

```json
{ "deleted": true, "trade": { "id": "…", "ticker": "0700.HK", "type": "sell", … },
  "cash_reversed": -76800, "cash": 1234.5, "shares_after": 200, "journal_deleted": 0,
  "journal_review": { "kind": "sell_close", "date": "2026-10-23", "message": "…" } }
```

### PATCH /positions/{ticker}

Body: `{ "stop": 350, "target": 450 }`, `{ "exit_mode": "levels_only" }`, or both. Send any of the three fields; `null` clears a stop or target.

- Changes the stop and target in force on every open journal entry of the stock. The entry plan (`entry_stop`, `entry_target`) is never changed.
- Each change of the stop is added to `stop_history`, and of the target to `target_history`, with `source: "api"` (see `GET /watchlist`).
- `exit_mode` says how the holding's exits are watched:
  - `strategy`, the default: the strategy's exit and trim rules, and the stop and target, as for every holding;
  - `levels_only`: only the stop and target. The strategy sends no exit or trim signals for this holding;
  - `external`: managed elsewhere, such as by a broker's trailing stop. No exit or trim signals, and no alerts on the stop and target, which stay on the position.

  Add signals are not affected by any of them. The mode belongs to the holding: it needs no journal entry, and goes back to `strategy` once the holding is sold out. Signals already recorded stay as they are.
- Every field is checked before anything changes. No open position gives 404 `no_open_position`; an unknown `exit_mode` gives 400; 503 `not_available` means exit modes are not available yet, and nothing changed.
- Answers `{ ticker, market, stop, target }`, and `exit_mode` when it was sent.

### GET /alerts

Query: `market` (optional).

All of the user's alerts come in one answer (an account holds at most 500): `total` is their number and `next_offset` is null.

`from_plan` is true for an alert the website keeps on the planned stop or target of an open position. These alerts move when the plan changes; the website never changes alerts the user created.

```json
{ "alerts": [
  { "id": "…", "ticker": "AAPL", "market": "us", "direction": "below", "threshold": 210,
    "note": null, "from_plan": false, "triggered_at": null, "created_at": "…" }
], "total": 1, "next_offset": null }
```

### POST /alerts

Body: `{ "market": "us", "ticker": "AAPL", "direction": "below", "threshold": 210, "note": "…" }`.

- A stock not on the watchlist is first looked up at its market's quote source, as the website's alert form does: one it does not know gives 404 `unknown_ticker`. A stock on the watchlist is taken as it is.
- A duplicate gives 409 `exists`.
- At most 500 alerts per account (403 `limit`); the plan's own stop and target alerts are counted apart and never block one.
- Answers 201 with `{ id, ticker, market, direction, threshold, note }`.

### PATCH /alerts/{id}

Body: any of `{ "threshold": 205, "note": "…", "rearm": true }`. The id comes from `GET /alerts` or `POST /alerts`.

- `threshold` is a new level; `note` a new note, `null` or `""` to clear it; `rearm: true` watches a fired alert again, as the website's **Re-arm** (重新启用) does. Without any of them, 400.
- A new level or a re-arm starts the watch over: only prices after it count. A price already past the level fires on the next scan. A fired alert given a new level stays fired until it is re-armed.
- Each watch fires once: an alert that already fired today and is re-armed (with or without a new level) fires again, as a new signal, if the price crosses again the same day.
- An alert on the plan's stop or target (`from_plan: true`) can only be re-armed: a new level or note gives 409 `from_plan`. Change the level with `PATCH /positions/{ticker}`.
- A level that matches another active alert on the same ticker and direction gives 409 `exists`, and so does re-arming an alert whose level another active alert now has.
- Answers 200 with `{ "alert": { … } }`, as `GET /alerts` lists it. Sending only `rearm: true` for one that has not fired changes nothing and adds `"unchanged": true`.

### DELETE /alerts/{id}

Deletes one of the user's alerts, as the website does. Its fired signals stay in `GET /signals`, with `alert.id` null.

- An alert on the plan's stop or target gives 409 `from_plan`: clear the stop or target with `PATCH /positions/{ticker}` instead.
- Answers 200 with `{ "deleted": true, "alert": { … } }`, the alert as it was.

### GET /account

The cash the user tracks in each market, in that market's currency. `amount` is `null` where cash is not tracked; trades then move no cash there.

```json
{ "cash": { "hk": { "amount": 12000, "currency": "HKD" }, "us": { "amount": null, "currency": "USD" }, "cn": { "amount": 5000, "currency": "CNY" } } }
```

### PUT /cash/{market}

Body: `{ "cash": 12000 }`. Sets the balance to what the broker shows, as the cash box on the website does, after a deposit or withdrawal for example. `null` stops tracking cash in that market. Negative is allowed (a buy above the cash leaves it negative). Answers `{ market, cash, currency }`.

### GET /dividends

Query, all optional: `market`, `ticker`, `from` and `to` (ex-dividend dates), `limit` (1–500, default 100), `offset` (default 0). A `ticker` without `market` finds its market from the code, as for `GET /trades`.

Answers `{ dividends, total, next_offset }`, newest ex-dividend date first; `next_offset` is null on the last page. Each dividend: `{ id, ticker, market, ex_div_date, pay_date, dps, currency, shares, amount, cash_delta, note, recorded_at }`. `cash_delta` is what it added to the cash; `null` when cash was not tracked.

### POST /dividends

Body: `{ "market": "hk", "ticker": "700", "ex_div_date": "2026-05-15", "dps": 4.5, "shares": 100, "pay_date": "2026-06-01", "currency": "HKD", "cash_amount": 427.5, "note": "…" }`. `pay_date`, `currency` (default the market's), `cash_amount` and `note` are optional.

- Recorded as the website's dividends page records it. `cash_amount` is what arrived in the market's cash after any withholding tax; with it, the cash goes up by that much in the same step. Without it, no cash moves. In a market whose cash is not tracked, the dividend is saved and `cash_added` is false.
- Answers 201 with `{ dividend, cash_added, cash }`.
- `external_id` (optional, 1 to 100 characters, such as the broker's record of the payment) guards against recording it twice, as for trades: the same id sent again in the same market records nothing and answers 409 `exists`, with the dividend already recorded under it beside the error (`"dividend": { … }`). Without it, the same dividend sent twice is recorded twice.

### DELETE /dividends/{id}

Deletes a dividend recorded by mistake and takes back what it added to the cash. Answers `{ deleted: true, dividend, cash_reversed, cash }`.

### GET /events

The writes made through API keys and connected apps, newest first: the log Settings → AI apps & API shows. Read scope. Over MCP it is the `get_events` tool.

Query, all optional: `action` (one, or a comma list such as `trade.record,trade.delete`), `ticker`, `since` and `until` (times, such as `2026-11-12T00:00:00Z`), `limit` (1–500, default 100) and `offset`.

- `action` is what was done: `trade.record`, `trade.edit`, `trade.delete`, `watchlist.add`, `watchlist.remove`, `position.levels`, `position.exit_mode`, `alert.create`, `alert.edit`, `alert.delete`, `cash.set`, `dividend.record`, `dividend.delete`.
- `source` is how it came: `{ "type": "key", "id", "name" }` for an API key, `{ "type": "app", "id", "name" }` for an app connected over OAuth; `name` is null once the key or app is gone.
- `summary` says what changed in a few words (English), as the settings page does; `detail` is what was logged.

```json
{ "events": [
    { "id": "…", "action": "trade.record", "at": "2026-11-10T01:00:00Z", "ticker": "0700.HK", "market": "hk",
      "source": { "type": "key", "id": "…", "name": "Broker sync" }, "summary": "Buy 100 @ 380, 2026-11-10",
      "detail": { "ticker": "0700.HK", "market": "hk", "type": "buy", "price": 380, "quantity": 100, "traded_at": "2026-11-10" } }
  ], "total": 1, "next_offset": null }
```

## Errors

Every error has the same shape:

```json
{ "error": { "code": "oversell", "message": "…" } }
```

The `message` is written for the user; pass it on.

- Every answer has an `x-request-id` header. A 500 also gives it in the message and as `error.request_id`: quote it when reporting the problem.
- A 401 has `WWW-Authenticate: Bearer realm="Stock Compass API"`. A 401 or an unknown endpoint names this reference in its message.
- A path that exists under other methods gives 405 `method_not_allowed`, with an `Allow` header listing them.

| Status | Codes | Meaning |
| --- | --- | --- |
| 400 | `bad_request`, `bad_ticker` | A field is missing or invalid; the message says which. |
| 401 | `unauthorized` | No key, or a key Stock Compass does not know. |
| 401 | `key_revoked`, `key_expired` | The key was revoked, or has expired; the message gives the date. Create a new key. |
| 403 | `scope` | A read-only key tried to write, or a key without the kind of write the request needs; the message names the scope. |
| 403 | `pro_required` | The account is not on Pro. |
| 403 | `limit` | Watchlist or price-alert limit reached. |
| 404 | `not_found` | Unknown endpoint. |
| 405 | `method_not_allowed` | The endpoint takes other methods; `Allow` lists them. |
| 404 | `unknown_ticker` | The ticker is not at the market's quote source. |
| 404 | `not_on_watchlist` | Removing a stock that is not on the watchlist. |
| 404 | `no_open_position` | Setting levels on a stock that is not held. |
| 404 | `trade_not_found` | No trade of this user has that id. |
| 404 | `dividend_not_found` | No dividend of this user has that id. |
| 404 | `alert_not_found` | No price alert of this user has that id. |
| 404 | `strategy_not_found` | No strategy of this user has that id. |
| 409 | `exists` | Already on the watchlist, the same alert already exists, or a trade or dividend is already recorded with that `external_id` (the body also has the `trade` or `dividend`). |
| 409 | `from_plan` | Changing or deleting the alert on a plan's stop or target; change the plan with `PATCH /positions/{ticker}`. |
| 409 | `holding` | Removing a stock that is still held. |
| 409 | `not_on_watchlist` | Recording a trade in a stock that is not on the watchlist. |
| 409 | `oversell` | Selling more than was held on that date; or a correction or deletion that would leave a sale of shares not held. |
| 409 | `out_of_order` | Recording a trade that comes before the stock's later trades (by date, or by `executed_at` within a date), when it would change them (the body also has `later_trades`). |
| 422 | `no_recent_signal` | `trigger: signal` with no recent signal. |
| 429 | `rate_limited` | More than 60 requests in a minute; retry after 60 seconds. |
| 500 | `internal` | Something went wrong on the server. Try again later. |
| 503 | `not_available` | Recording dividends through the API is not switched on yet; record it on the website's Dividends page. |

Any request can also give 401, 403 `pro_required`, 429 and 500. Besides those:

| Endpoint | Can give |
| --- | --- |
| `GET /me` | none |
| `GET /watchlist` | 400 `bad_request` |
| `POST /watchlist` | 400 `bad_request` or `bad_ticker`, 403 `scope` or `limit`, 404 `unknown_ticker`, 409 `exists` |
| `DELETE /watchlist/{ticker}` | 400, 403 `scope`, 404 `not_on_watchlist`, 409 `holding` |
| `GET /signals` | 400 `bad_request` |
| `GET /trades` | 400 `bad_request` or `bad_ticker` |
| `GET /journal` | 400 `bad_request` or `bad_ticker` |
| `GET /journal/stats` | 400 `bad_request` |
| `GET /strategies` | 400 `bad_request` |
| `GET /strategies/{id}` | 400 `bad_request`, 404 `strategy_not_found` |
| `POST /trades` | 400, 403 `scope` or `limit`, 404 `unknown_ticker`, 409 `exists`, `not_on_watchlist`, `out_of_order` or `oversell`, 422 `no_recent_signal`, 503 `not_available` |
| `POST /trades/batch` | 400 `bad_request`, 403 `scope`; each trade's own refusal is in its result |
| `POST /reconcile/positions` | 400 `bad_request` or `bad_ticker` |
| `GET /settings/trading-costs` | 400 `bad_request` |
| `PATCH /trades/{id}` | 400, 403 `scope`, 404 `trade_not_found`, 409 `oversell` |
| `DELETE /trades/{id}` | 400, 403 `scope`, 404 `trade_not_found`, 409 `oversell` |
| `PATCH /positions/{ticker}` | 400, 403 `scope`, 404 `no_open_position`, 503 `not_available` |
| `GET /alerts` | 400 `bad_request` |
| `POST /alerts` | 400, 403 `scope` or `limit`, 404 `unknown_ticker`, 409 `exists` |
| `PATCH /alerts/{id}` | 400, 403 `scope`, 404 `alert_not_found`, 409 `from_plan` or `exists` |
| `DELETE /alerts/{id}` | 400, 403 `scope`, 404 `alert_not_found`, 409 `from_plan` |
| `GET /account` | none |
| `PUT /cash/{market}` | 400, 403 `scope` |
| `GET /dividends` | 400 `bad_request` or `bad_ticker` |
| `GET /events` | 400 `bad_request` or `bad_ticker` |
| `POST /dividends` | 400 `bad_request` or `bad_ticker`, 403 `scope`, 409 `exists`, 503 `not_available` |
| `DELETE /dividends/{id}` | 400, 403 `scope`, 404 `dividend_not_found`, 503 `not_available` |

## What the website shows the user

Every write made through a key or a connection is logged, with what changed, the stock and the time. The user sees the log in Settings → AI apps & API, and a client reads it with `GET /events`; which also lists the active keys and connected apps. The website refreshes the watchlist, cash, signals and journal when the user opens it or switches back to it.

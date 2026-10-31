# Connecting to Stock Compass: OAuth, API keys, MCP and REST

Read this when the user wants to connect an AI app, an agent or a script to Stock Compass, or asks about API keys, rate limits or the REST endpoints.

**Labels.** Buttons are named as the English site shows them, followed by the Chinese site's label in brackets. When you point the user to one, use the label for the language their site is in.

All access is to the signed-in user's own records, and needs a Stock Compass **Pro** account (or an admin account). The plan is checked on every call, so access stops when Pro lapses.

## Two ways in

| | OAuth connection | Personal API key |
| --- | --- | --- |
| Best for | AI apps that support MCP with OAuth (Claude, ChatGPT, many others) | Scripts, command-line agents, and apps that can only send a fixed header |
| How it starts | The app opens stock-compass.app; the user signs in and approves | The user creates a key in Settings → AI apps & API (设置 → AI 应用与 API) |
| Secret the user handles | None | The key, shown once |
| Read only or read and write | Chosen on the approval page; read only by default | Chosen when the key is created: **Allow writes** (允许写入) |
| Works with | `/api/mcp` | `/api/mcp` and `/api/v1` (REST) |
| Ends | Disconnect in Settings → AI apps & API | Revoke in the same place, or the key's expiry |

Both allow 60 requests a minute. The count is kept separately for each key and each connected app, not for the account: two keys of one user each get 60. Over MCP every message counts, `initialize` and `tools/list` included, and a write takes two calls (a preview, then the change), so recording 30 trades takes 60 or more. Space a long list of writes about a second apart. Over the limit, the answer is HTTP 429 with `Retry-After: 60`.

## Quick start (REST)

1. Create a key on the site: Settings → AI apps & API (设置 → AI 应用与 API), for example https://stock-compass.app/hk/settings/api-keys. Tick **Allow writes** (允许写入) to try a write.
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
- When creating a key, the user gives it a name (up to 60 characters) and picks an expiry: 30 days, 90 days, 1 year, or never. Writes are allowed only if **Allow writes** (允许写入) is ticked.
- A user can have at most 5 active keys. Revoking a key takes effect at once.
- Send the key as a header on every request: `Authorization: Bearer sc_…`.
- Treat a key as a password. Keep it in the agent's or the script's own secret settings, never in a conversation, a shared file or a repository. If it leaks, revoke it.

## MCP

The MCP endpoint is `https://stock-compass.app/api/mcp`. It uses Streamable HTTP, is stateless and answers in JSON.

- **With OAuth**: add the address as a remote MCP server and choose OAuth. The client finds the sign-in details itself. A request without a token gets HTTP 401, with a `WWW-Authenticate` header that points to `https://stock-compass.app/.well-known/oauth-protected-resource/api/mcp`.
- **With an API key**: add the address as a remote MCP server and set the header `Authorization: Bearer sc_…`. For example, in Claude Code:
  ```bash
  claude mcp add --scope user --transport http stock-compass https://stock-compass.app/api/mcp --header "Authorization: Bearer sc_…"
  ```
  `--scope user` makes it available in every folder; without it, only in the one the command ran in. Other clients have their own setting for request headers.
- **For client developers**:
  - Clients register themselves (dynamic client registration). They are public clients, with no secret, and PKCE with `S256` is required. Redirect URIs (one to ten) are https, http on localhost, or an app scheme, with no fragment. Registrations from sites other than the known apps are limited to 10 an hour from one address.
  - An access token lasts an hour. A refresh token lasts 90 days and is replaced at each use; an old one sent again revokes that connection's tokens. So a connection unused for 90 days has to be approved again.
  - A request with no scope asks for both read and write; the user still chooses on the approval page.
  - MCP protocol versions: 2025-11-25, 2025-06-18 and 2025-03-26.
  - `/api/v1` sends no CORS headers: an API key does not belong in a web page.

### Connecting common apps

| App | How to connect |
| --- | --- |
| Claude (web, desktop app, Cowork) | Install the Stock Compass plugin: Customize → Plugins → Add → Add marketplace → `Stock-Compass/stock-compass-plugin`, then connect it on the plugin's Connectors tab. Or add the address directly as a custom connector. |
| Claude Code | `claude plugin marketplace add Stock-Compass/stock-compass-plugin`, then `claude plugin install stock-compass@stock-compass`; in Claude Code, `/reload-plugins`, then `/mcp` to sign in. Or `claude mcp add --scope user --transport http stock-compass` with the address. |
| ChatGPT | Start Pro or the free trial on stock-compass.app first: ChatGPT's approval page cannot start one. Then chatgpt.com/plugins → + → Add custom MCP server. Paste the address and choose OAuth. |
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
| `add_to_watchlist` | `POST /watchlist` |
| `remove_from_watchlist` | `DELETE /watchlist/{ticker}` |
| `record_trade` | `POST /trades` |
| `edit_trade` | `PATCH /trades/{id}` |
| `delete_trade` | `DELETE /trades/{id}` |
| `set_stop_target` | `PATCH /positions/{ticker}` |
| `create_price_alert` | `POST /alerts` |

Tool arguments use the same names as the REST fields. In MCP, `types` for `get_signals` is a list. Tool errors come back as a result marked as an error, with the message described under "Errors" below.

**Writes over MCP take two calls.** Sent without `preview_id`, a write tool changes nothing. It answers with `status: "confirmation_needed"`, the `change` as it would be made (for `record_trade`, with the date it defaults to), for `edit_trade` and `delete_trade` the `trade` it changes, and a `preview_id` (`pv_` and 12 hex digits). Once the user agrees, the same arguments sent again with that `preview_id` make the change. A `preview_id` fits only that change on that connection, and lasts 10 to 20 minutes. Sent with any other change, or late, it changes nothing and comes back with a new preview. `record_trade` also returns a `note` that no order is placed. A preview does not check that the change can be made (the stock on the watchlist, enough shares to sell, the plan's limits): the second call does, so look at `get_watchlist` first. The REST API has no previews.

## REST API

Base URL: `https://stock-compass.app/api/v1`. It accepts **API keys only**; OAuth tokens work only with `/api/mcp`.

- Requests and responses are JSON.
- `GET` needs the read scope; `POST`, `PATCH` and `DELETE` need write.
- `market` is `hk`, `us` or `cn`.
- New fields and parameters are optional, so a client written for an earlier version keeps working.

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
- A trade has a date but no time. Trades on the same date count in the order they were recorded: that order decides the holding on the day, whether a sell is covered, and which journal entries a sale closes. Record a day's trades in the order they happened.
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

What the key can do.

```json
{ "user_id": "…", "scopes": ["read", "write"], "requests_per_minute": 60 }
```

### GET /watchlist

Query, all optional:
- `market`;
- `holding`: `true` for holdings only, `false` for stocks not held;
- `limit` (1–1000) and `offset`: one page. Without `limit`, the whole list.

`total` is the number of stocks that match, before paging. `limit` and `offset` come back when a page was asked for.

`stop` and `target` are the ones in force on the open position; they are null when the stock is not held. A holding also has:
- `opened_at`: the date of the oldest open journal entry, when the position was opened;
- `journal_entry_id`, `entry_stop`, `entry_target`: the latest buy's journal entry, whose levels are in force, with the stop and target set when it was bought;
- `stop_history`: every change of that entry's stop since, oldest first, from any source (website, API or app). `from` or `to` is null when there was no stop. Target changes are not kept.

```json
{ "items": [
  { "ticker": "0700.HK", "market": "hk", "name": "TENCENT", "name_zh": "腾讯控股",
    "holding": true, "quantity": 200, "cost_price": 382.4, "stop": 365, "target": 450,
    "opened_at": "2026-10-02", "journal_entry_id": 42, "entry_stop": 350, "entry_target": 450,
    "stop_history": [ { "at": "2026-10-15T08:10:00Z", "from": 350, "to": 358 },
                      { "at": "2026-10-21T08:05:00Z", "from": 358, "to": 365 } ],
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

Query:
- `market` (optional);
- `days`: 0–90, default 7;
- `types`: comma-separated, any of `buy,add,exit,trim,price_alert`; default `buy,add,exit,trim`;
- `state`: `all` (the default), `new` or `active`. See runs below.

Each market's buy, add, exit and trim signals come from the strategy that was active in that market on each date. Price-alert signals are the user's own alerts that fired. Results are newest first.

- `date` is the daily bar the signal was computed on, in the market's time zone. `recorded_at` is when the scan wrote it (UTC).
- Buy and add wait for that bar to be final: they are recorded by the next trading day's 09:00 scan, in that market's time, so a Friday's appear on Monday. Exit and trim are recorded during the session, at the price when the scan ran.
- `days` counts back from the market's own date (for all markets, the earliest of them), not from UTC's.
- The signal says which strategy and type fired, not which condition of the rule.

**Runs.** A rule that stays true is recorded again on every bar: an exit rule that holds for a week gives five exit signals. Signals of one strategy, ticker and type on consecutive scanned bars form one run.
- `active_since` is the date of the run's first bar, and `occurrences` how many bars of the run there have been up to this signal (1 on the first).
- A run ends on a bar the scan evaluated without recording it. A day with no scan, such as an exchange holiday, does not end it. When the account has no scan records for the market in the period, weekdays are used instead, and a holiday then ends a run.
- Runs are traced back up to 30 days before `from`.
- `state=new` returns only the first bar of each run, `state=active` only the later ones. A price-alert signal is always a run of one.

```json
{ "from": "2026-10-18",
  "note": "Buy and add signals are recorded once their daily bar is final …",
  "signals": [
    { "id": 123, "ticker": "ANF", "market": "us", "type": "exit", "date": "2026-10-23",
      "price": 97.2, "strategy": "Trend", "recorded_at": "2026-10-23T17:05:00Z",
      "active_since": "2026-10-19", "occurrences": 5 }
  ] }
```

### GET /trades

Query: `market`, `ticker`, `from`, `to`, `external_id` (all optional); `limit` (1–500, default 100); `offset` (default 0). A `ticker` without `market` finds its market from the code: `700` is 0700.HK, `600519` is 600519.SS, `BRK.B` is BRK-B.

Results are newest first. `external_id` is null for a trade recorded without one, such as on the website. `fees` is the fee the client reported, with `fees_source: "provided"`; it is null with `"estimated"` when none was. `estimated_fees` is then the estimate from the user's trading-cost settings that the trade was recorded with, as the website's P&L counts it (for an old trade with none kept, from the current settings); it is null when a fee was reported.

```json
{ "trades": [
    { "id": "…", "ticker": "0700.HK", "market": "hk", "type": "buy", "price": 382.4,
      "quantity": 200, "traded_at": "2026-10-23", "recorded_at": "2026-10-23T03:00:00Z",
      "external_id": "FUTU-8231", "fees": 3.86, "fees_source": "provided", "estimated_fees": null }
  ], "limit": 100, "offset": 0 }
```

### POST /trades

Records a buy or sell the user made, exactly as the trade form on the website does. Cash, the holding, the lots and the trade journal all move together, in one transaction.

| Field | Required | Notes |
| --- | --- | --- |
| `market`, `ticker` | yes | The stock must be on the watchlist (409 `not_on_watchlist`), unless `add_to_watchlist` is true. |
| `type` | yes | `buy` or `sell`. |
| `price` | yes | Per share, without fees. |
| `quantity` | yes | Whole shares. |
| `traded_at` | no | The trade date in that market. Defaults to today there; cannot be in the future. |
| `trigger` | buy only | `signal`, `manual` or `other`. See below. |
| `stop`, `target` | buy only | The planned stop and target. |
| `thesis` | buy only | Up to 1000 characters. |
| `close_reason` | sell only | `rule_exit`, `rule_trim`, `manual_stop`, `manual_target` or `other`. |
| `lesson` | sell only | Up to 1000 characters. |
| `add_to_watchlist` | no | `true`: for a buy of a stock not on the watchlist, add it first, as `POST /watchlist` does. Default `false`. |
| `external_id` | no | The client's own id for the trade, such as the broker's order number: 1 to 100 characters. See below. |
| `fees` | no | What the broker charged for this trade, zero or more, in its currency. Without it, estimated from the user's settings. |

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
- **`external_id`** guards against recording a trade twice. It is unique per user and market.
  - Sent again, the same id records nothing and answers 409 `exists`, with the trade already recorded under it beside the error: `{ "error": { … }, "trade": { "id": "…", … } }`. That is so whatever else the request says.
  - So a request that timed out can be sent again as it was. Two requests at once with the same id record one trade.
  - Once that trade is deleted, the id is free again.
  - Without `external_id`, sending the same trade twice records it twice.
- **Same-day order**: trades on one date count in the order they are recorded. See [Dates and times](#dates-and-times).

Answers 201:

```json
{ "trade_id": "…", "ticker": "0700.HK", "market": "hk", "type": "buy", "price": 382.4, "quantity": 200,
  "traded_at": "2026-10-23", "external_id": "FUTU-8231", "fees": 3.86, "fees_source": "provided", "shares_before": 0, "shares_after": 200, "cash": 12345.6,
  "journal_entry_id": 42, "trigger": { "type": "signal", "signal_id": 123, "signal_date": "2026-10-22" } }
```

For a sale, `journal_closed_shares` takes the place of `journal_entry_id` and `trigger`; `ignored` and `journal_note` appear only when `close_reason` and `lesson` were not kept. `external_id` is there only when one was sent.

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

Body: `{ "stop": 350, "target": 450 }`. Send either field or both; `null` clears it.

- Changes the stop and target in force on every open journal entry of the stock. The entry plan (`entry_stop`, `entry_target`) is never changed.
- Each change of the stop is added to `stop_history` (see `GET /watchlist`).
- No open position gives 404 `no_open_position`.
- Answers `{ ticker, market, stop, target }`.

### GET /alerts

Query: `market` (optional).

`from_plan` is true for an alert the website keeps on the planned stop or target of an open position. These alerts move when the plan changes; the website never changes alerts the user created.

```json
{ "alerts": [
  { "id": "…", "ticker": "AAPL", "market": "us", "direction": "below", "threshold": 210,
    "note": null, "from_plan": false, "triggered_at": null, "created_at": "…" }
] }
```

### POST /alerts

Body: `{ "market": "us", "ticker": "AAPL", "direction": "below", "threshold": 210, "note": "…" }`.

- A duplicate gives 409 `exists`.
- At most 500 alerts per account (403 `limit`).
- Answers 201 with `{ id, ticker, market, direction, threshold, note }`.

## Errors

Every error has the same shape:

```json
{ "error": { "code": "oversell", "message": "…" } }
```

The `message` is written for the user; pass it on.

| Status | Codes | Meaning |
| --- | --- | --- |
| 400 | `bad_request`, `bad_ticker` | A field is missing or invalid; the message says which. |
| 401 | `unauthorized` | No key, or a key that is unknown, revoked or expired. |
| 403 | `scope` | A read-only key tried to write. |
| 403 | `pro_required` | The account is not on Pro. |
| 403 | `limit` | Watchlist or price-alert limit reached. |
| 404 | `not_found` | Unknown endpoint. |
| 404 | `unknown_ticker` | The ticker is not at the market's quote source. |
| 404 | `not_on_watchlist` | Removing a stock that is not on the watchlist. |
| 404 | `no_open_position` | Setting levels on a stock that is not held. |
| 404 | `trade_not_found` | No trade of this user has that id. |
| 409 | `exists` | Already on the watchlist, the same alert already exists, or a trade is already recorded with that `external_id` (the body also has the `trade`). |
| 409 | `holding` | Removing a stock that is still held. |
| 409 | `not_on_watchlist` | Recording a trade in a stock that is not on the watchlist. |
| 409 | `oversell` | Selling more than was held on that date; or a correction or deletion that would leave a sale of shares not held. |
| 422 | `no_recent_signal` | `trigger: signal` with no recent signal. |
| 429 | `rate_limited` | More than 60 requests in a minute; retry after 60 seconds. |
| 500 | `internal` | Something went wrong on the server. Try again later. |

Any request can also give 401, 403 `pro_required`, 429 and 500. Besides those:

| Endpoint | Can give |
| --- | --- |
| `GET /me` | none |
| `GET /watchlist` | 400 `bad_request` |
| `POST /watchlist` | 400 `bad_request` or `bad_ticker`, 403 `scope` or `limit`, 404 `unknown_ticker`, 409 `exists` |
| `DELETE /watchlist/{ticker}` | 400, 403 `scope`, 404 `not_on_watchlist`, 409 `holding` |
| `GET /signals` | 400 `bad_request` |
| `GET /trades` | 400 `bad_request` or `bad_ticker` |
| `POST /trades` | 400, 403 `scope` or `limit`, 404 `unknown_ticker`, 409 `exists`, `not_on_watchlist` or `oversell`, 422 `no_recent_signal` |
| `PATCH /trades/{id}` | 400, 403 `scope`, 404 `trade_not_found`, 409 `oversell` |
| `DELETE /trades/{id}` | 400, 403 `scope`, 404 `trade_not_found`, 409 `oversell` |
| `PATCH /positions/{ticker}` | 400, 403 `scope`, 404 `no_open_position` |
| `GET /alerts` | 400 `bad_request` |
| `POST /alerts` | 400, 403 `scope` or `limit`, 409 `exists` |

## What the website shows the user

Every write made through a key or a connection is logged, with what changed, the stock and the time. The user sees the log in Settings → AI apps & API, which also lists the active keys and connected apps. The website refreshes the watchlist, cash, signals and journal when the user opens it or switches back to it.

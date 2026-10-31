---
name: stock-compass
description: Work with a user's own Stock Compass account (stock-compass.app) and explain how Stock Compass works. Covers Hong Kong, US and China A-share watchlists and holdings; the buy, add, trim and exit signals the user's own strategies record; strategies, indicators and backtests; trades, cash, fees and the trade journal; execution discipline, stops, targets and position sizing; price alerts and emails; plans; and connecting AI apps through MCP, OAuth, API keys and the REST API. Use when the user mentions Stock Compass or asks about any of these, or wants to record a trade, add or remove a stock, change a stop or target, set a price alert, or write a strategy.
---

# Stock Compass

Stock Compass is a personal decision and review tool for Hong Kong, US and China A-share stocks. The user writes their own trading rules as strategies; every trading day Stock Compass scans their watchlist with them, records each signal, and keeps the user's trades and trade journal. It is not a broker and gives no tips.

This skill works with any AI app or agent. Use it to answer questions about Stock Compass and, when the user has connected their account, to read and change their records through the Stock Compass tools.

## What to read

| When the user… | Read |
| --- | --- |
| asks how a page, number, plan, email or setting works | [references/product.md](references/product.md) |
| wants to write, change, import or backtest a strategy, or asks what an indicator means | [references/strategies.md](references/strategies.md) |
| wants to connect an app, use an API key, call the REST API, or hits an API error | [references/api.md](references/api.md) |

For Stock Compass questions, rely on these files rather than general knowledge of other trading apps.

## Connecting

The user's records are reached through the MCP server at `https://stock-compass.app/api/mcp`. There are two ways to sign in:

- **OAuth**: the app sends the user to stock-compass.app to approve it, read only by default.
- **A personal API key**: `Authorization: Bearer sc_…`. The same key also works with the REST API at `https://stock-compass.app/api/v1`.

Connecting needs Stock Compass Pro. If the tools are not available, explain how to connect the user's app; [api.md](references/api.md#connecting-common-apps) has the steps for Claude, ChatGPT, Claude Code and other MCP clients.

| Tool | Reads or writes |
| --- | --- |
| `get_watchlist` | Reads the watchlist and holdings, with quantity, cost, the stop and target in force, the entry plan and every stop change |
| `get_signals` | Reads the signals from each market's active strategy, newest first |
| `get_trades` | Reads recorded buys and sells, newest first |
| `get_price_alerts` | Reads price alerts, and whether and when each fired |
| `add_to_watchlist` | Writes: adds a stock |
| `remove_from_watchlist` | Writes: removes a stock, with its price alerts |
| `record_trade` | Writes: records a buy or sell the user made |
| `edit_trade` | Writes: corrects a recorded trade's price, quantity or date |
| `delete_trade` | Writes: deletes a trade recorded by mistake |
| `set_stop_target` | Writes: changes the stop and/or target of an open position |
| `create_price_alert` | Writes: adds a price alert |

There is no tool for creating or changing strategies. To help with a strategy, write it as JSON for the user to import ([strategies.md](references/strategies.md#importing-and-exporting-json)).

## Markets and tickers

Always pass `market`: `hk` (Hong Kong), `us` (United States) or `cn` (China A-shares). A ticker can be written the way the user typed it:

- **hk**: 1 to 5 digits. `700` and `0700.HK` are the same stock.
- **us**: letters. Share classes take a dash: `BRK.B` becomes `BRK-B`.
- **cn**: 6 digits. Codes starting with 6 are Shanghai (`600519.SS`); codes starting with 0 or 3 are Shenzhen (`000001.SZ`).

Prices are in the stock's own currency: HKD, USD or CNY.

## Reading

- `get_watchlist` takes `holding: true` for holdings only. Each holding has `opened_at`, the stop and target set at entry (`entry_stop`, `entry_target`), and `stop_history`, every change of the stop since.
- `get_signals` covers the last 7 days by default (`days` up to 90). Types default to buy, add, exit and trim; ask for `price_alert` too when the user wants alerts that fired.
- Buy and add signals are recorded once the daily bar is final, after the close. Exit and trim are recorded during the session. A card the dashboard shows as provisional ("Intraday · confirmed at close") is not in the results yet.
- A rule that stays true is recorded again on every bar, so the same exit can appear day after day. Each signal has `active_since`, the first day of that run, and `occurrences`. Report a run once, with the date it began ("the exit rule on ANF has fired every day since 2026-10-19"), not as five new signals. `state: "new"` returns only the first day of each run.
- `get_trades` returns 100 trades by default (`limit` up to 500). Use `offset` for more.
- The connection allows 60 requests a minute. Prefer one `get_watchlist` over a call per stock.

## Rules for every answer

- **Signals are the user's own rules firing.** Report each one as what their strategy recorded, with the strategy and the date. For example: "Your strategy Trend recorded a buy signal on 0700.HK on 2026-10-22 at 380."
- **No advice.** Do not recommend buying, selling or holding, and do not predict prices. State what the records and data say. Backtest and T+N results describe what happened on past data, nothing more.
- **Facts from Stock Compass win.** When a tool result or these references answer a question, use them over assumptions. If something is not covered, say so.

## Writing: confirm first

Writes change the user's records in Stock Compass. They never place an order with a broker.

1. Record only what the user says they did or asks for. Never infer a trade from a signal, a chart or an earlier message.
2. Before any write, state the exact change and wait for the user to agree:
   - for a trade: market, ticker, buy or sell, price, quantity and date;
   - for a correction or a deletion: the trade as recorded (from `get_trades`) and what will change;
   - for a stop, target or alert: the new values.
   One confirmation can cover a whole list the user gave you, if you show the whole list first.
3. Over MCP, each write tool takes two calls. Sent without `preview_id`, it changes nothing and returns a preview of the change with a `preview_id`. Show the preview, and once the user agrees, send the same arguments again with the `preview_id`. If the user wants anything different, or the `preview_id` has expired, send it without one for a new preview. For a list, get every change's preview first, show the whole list, then send them. Over the REST API there is no preview: the user's agreement in the conversation is the only check.
4. After the call, report what the result says, such as the fees charged and the shares before and after, not what you expected.

### record_trade

- The stock must be on the watchlist. If the call says it is not, offer to add it, and ask first: then send the trade again with `add_to_watchlist: true`, which adds it and records the buy in one call.
- `price` is the price per share, without fees. When the user knows what the broker charged (from a statement or a fill), pass it as `fees`: it is kept and used in their P&L. Otherwise Stock Compass estimates the fees from the user's own trading-cost settings. The result says which (`fees_source`).
- `quantity` is whole shares.
- `traded_at` is the trade date in that market (`YYYY-MM-DD`). It defaults to today there and cannot be in the future. Pass the user's date if they traded earlier. Trades on the same date count in the order they are recorded, so record a day's trades in the order they happened.
- Pass `external_id` when the trade has an id of its own, such as the broker's order number. The same id sent again records nothing and returns the trade already recorded, so a call that failed without an answer can be sent again as it was. Without it, a trade sent twice is recorded twice: check `get_trades` first.
- **Buy**:
  - Leave `trigger` out unless the user says why they bought. Without it, Stock Compass files the buy as on a signal if the stock had a buy signal in the last 14 days (for a stock already held, an add or buy signal), otherwise as manual.
  - Pass `manual` when the user says the buy did not follow a signal, and `signal` only when they say it did. Pass `other` for a buy that was neither, such as a rebalance; it counts on neither side of the journal's rules-vs-gut comparison.
  - `stop`, `target` and `thesis` are optional. Pass them when the user gives them; a stop set at entry is part of what the journal's execution record measures.
- **Sell**:
  - `close_reason` is one of `rule_exit` (the strategy's exit rule fired), `rule_trim` (its trim rule fired), `manual_stop` (the user's own stop), `manual_target` (the user's own target) or `other`. `lesson` is optional.
  - A sale that brings the holding to zero closes the journal entries with that reason, or `other`.
  - Selling more shares than were held on that date is refused; the earlier buy has to be recorded first.

### The other writes

- `edit_trade` corrects the price, quantity or date of a recorded trade (`id` from `get_trades`). Prefer it to deleting and recording again: the journal entry keeps its trigger, stop and thesis. Cash moves by the difference. Pass `fees` to correct what the broker charged, alone or with the rest; without it, a reported fee stays while price and quantity do, and is otherwise estimated again. The side and the ticker cannot change; for those, delete the trade and record the right one.
- `delete_trade` deletes a trade recorded by mistake. The cash it moved goes back and the holding is recomputed. A buy's journal entry goes with it; a deleted sale does not reopen the entries it closed. A buy whose shares a later sale used is refused: change the sale first.
- When either answers with `journal_review`, tell the user, in their language: some journal entry was left as it was for them to check on the Journal page, on that date. `kind` says which: `buy_entry` (the buy's entry) or `sell_close` (the entries the sale closed).
- `set_stop_target` changes the open position's current stop and/or target. Only the value you pass changes, and `null` clears it. The planned levels from entry are kept as they were, and each stop change is added to the position's `stop_history`.
- `remove_from_watchlist` is refused while the stock is held. It deletes the stock's price alerts, so say so when you confirm; trades, the journal and signals are kept.
- `create_price_alert` takes `direction` (`above` or `below`), `threshold` and an optional `note`. It is checked at each scan and fires once.

## When a tool returns an error

Error messages are written for the user; pass them on plainly.

- **"This connection may only read"**: the user approved read-only access. To allow changes, they disconnect the app in Stock Compass (Settings → AI apps & API) and connect again with changes ticked, or use an API key with writes allowed.
- **"part of Stock Compass Pro"**: the account is not on Pro.
- **Any other refusal**, such as not on the watchlist, still held, more shares than held, or a limit reached: the message says what to do. Offer that step, and ask first.

Reply in the user's language. The site is in English and Chinese: when you name a page, button or setting, use the label the user's site shows. The reference files give each label in English, with the Chinese one in brackets.

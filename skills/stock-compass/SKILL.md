---
name: stock-compass
description: Work with the user's own Stock Compass records through the Stock Compass connector. These include their Hong Kong, US and China A-share watchlist and holdings, the buy, add, trim and exit signals their own strategies recorded, their trades and trade journal, stops and targets, and price alerts. Use when the user mentions Stock Compass, asks about their watchlist, holdings, positions, signals, stops, targets or price alerts, or wants to record a buy or sell, add or remove a stock, change a stop or target, or set a price alert.
---

# Stock Compass

Stock Compass (stock-compass.app) scans the user's watchlist every trading day with strategies the user wrote, and keeps their trades, holdings and trade journal. The `stock-compass` connector gives you their records. You see only what belongs to the signed-in user.

If the connector's tools are not available, tell the user how to connect it:

- **claude.ai or the desktop app**: Customize → Plugins → Stock Compass → Connectors, then connect and sign in.
- **Claude Code**: run `/mcp` and sign in to `stock-compass`.

The connector needs a Stock Compass Pro account.

## Tools

| Tool | Reads or writes | What it does |
| --- | --- | --- |
| `get_watchlist` | Reads | The watchlist, with whether each stock is held, the quantity and cost price, and the stop and target in force |
| `get_signals` | Reads | Signals recorded by the strategy active in each market, newest first |
| `get_trades` | Reads | Recorded buys and sells, newest first |
| `get_price_alerts` | Reads | Price alerts, and whether and when each fired |
| `add_to_watchlist` | Writes | Adds a stock |
| `remove_from_watchlist` | Writes | Removes a stock, with its price alerts |
| `record_trade` | Writes | Records a buy or sell the user made |
| `set_stop_target` | Writes | Changes the stop and/or target of an open position |
| `create_price_alert` | Writes | Adds a price alert |

## Markets and tickers

Always pass `market`: `hk` (Hong Kong), `us` (United States) or `cn` (China A-shares). The ticker can be written the way the user typed it:

- **hk**: 1 to 5 digits. `700` and `0700.HK` both mean `0700.HK`.
- **us**: letters. Share classes take a dash: `BRK.B` becomes `BRK-B`.
- **cn**: 6 digits. Codes starting with 6 are Shanghai (`600519.SS`); codes starting with 0 or 3 are Shenzhen (`000001.SZ`).

Prices are in the stock's own currency: HKD, USD or CNY.

## Reading

- `get_signals` covers the last 7 days by default (`days` up to 90). Types default to buy, add, exit and trim. Ask for `price_alert` too when the user wants alerts that fired.
- Buy and add signals are recorded once the daily bar is final, after the close. Exit and trim are recorded during the session. A card that the Stock Compass dashboard shows as provisional during the session is not in the results yet.
- `get_trades` returns 100 trades by default (`limit` up to 500). Use `offset` for the next page.
- The connection allows 60 requests a minute. Prefer one `get_watchlist` over one call per stock.

## Reporting signals and data

Signals are what the user's own strategy rules produced, so attribute them to the strategy. For example: "Your strategy Trend recorded a buy signal on 0700.HK on 2026-10-22 at 380."

State what the records say. Do not recommend buying, selling or holding, and do not predict prices.

## Writing: confirm first

Writes change the user's records in Stock Compass. They never place an order with a broker.

1. Record only what the user says they did or asks for. Never infer a trade from a signal, a chart or an earlier message.
2. Before any write tool, state the exact change and wait for the user to agree:
   - for a trade: market, ticker, buy or sell, price, quantity and date;
   - for a stop or target, or an alert: the new values.
   One confirmation can cover a whole list the user gave you, if you show the whole list first.
3. After the call, report what the result says, such as the fees charged and the shares before and after, not what you expected.

### record_trade

- The stock must be on the watchlist. If the call says it is not, offer to add it with `add_to_watchlist`, and ask first.
- `price` is the price per share, without fees. Stock Compass computes the fees from the user's own trading-cost settings and returns them.
- `quantity` is whole shares.
- `traded_at` is the trade date in that market, `YYYY-MM-DD`. It defaults to today there and cannot be in the future. Pass the user's date if they traded on an earlier day.
- **Buy**:
  - Leave `trigger` out unless the user says why they bought. Without it, Stock Compass files the buy as on a signal when the stock had a buy signal in the last 14 days (for a stock already held, an add or buy signal), otherwise as manual.
  - Pass `manual` when the user says the buy did not follow a signal, and `signal` only when they say it did.
  - `stop`, `target` and `thesis` are optional. Pass them when the user gives them.
- **Sell**:
  - `close_reason` is one of `rule_exit`, `rule_trim`, `manual_stop`, `manual_target` or `other`. `lesson` is optional.
  - A sale that brings the holding to zero closes the journal entries with that reason, or `other` if none is given.
  - A sale of more shares than were held on that date is refused. The earlier buy has to be recorded first.

### The other writes

- `set_stop_target` changes the open position's stop and/or target. Only the value you pass changes, and `null` clears it. There must be an open position.
- `remove_from_watchlist` is refused while the stock is held. It also deletes the stock's price alerts, so say so when you confirm.
- `create_price_alert` takes `direction` (`above` or `below`), `threshold` (a price) and an optional `note`.

## When a tool returns an error

Error results are written for the user, so pass them on plainly.

- **"This connection may only read"**: the user approved read-only access. To allow changes, they disconnect the app in Stock Compass (Settings → AI apps & API) and connect again with changes ticked.
- **"part of Stock Compass Pro"**: the account is not on Pro.
- **Any other refusal**, such as not on the watchlist, still held, more shares than held, or a limit reached, says what to do. Offer that step and ask first.

Reply in the user's language. In Chinese, the markets are 港股, 美股 and A股.

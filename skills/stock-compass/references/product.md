# Stock Compass: how the product works

Read this when the user asks what Stock Compass does, how a page, number or email works, what their plan includes, or how their data is handled. Strategy rules, indicators and backtests are in [strategies.md](strategies.md). Connecting apps, API keys and the REST API are in [api.md](api.md).

**Labels.** The site is in English and Chinese. Pages, buttons and messages are named as the English site shows them, followed by the Chinese site's label in brackets: **Record buy** (记录买入). When you point the user to something, use the label for the language their site is in.

## What it is, and what it is not

Stock Compass (https://stock-compass.app, English and Chinese) is a personal decision and review tool for Hong Kong, US and China A-share stocks. In it, the user:

- writes their own buy, add, trim and exit rules as a strategy;
- backtests the strategy on real daily prices;
- lets it scan their watchlist every trading day.

Stock Compass records every signal, scores it against what the price did next, and keeps the user's trades and trade journal. It then shows whether each trade followed the rules or not.

It is **not**:
- a broker: no brokerage connection, no orders;
- a source of tips: signals are the user's own rules firing, not recommendations;
- a real-time terminal: intraday prices are delayed 15–20 minutes.

The product avoids judgement words. Its data summaries end with "The above is a summary of data, not investment advice." (以上为数据汇总，不构成投资建议。)

## Markets, tickers and data

| Market | Code | Currency | Ticker form |
| --- | --- | --- | --- |
| Hong Kong (港股) | `hk` | HKD | 1–5 digits, stored as `0700.HK` |
| China A-shares (A股) | `cn` | CNY | 6 digits: 6xxxxx → `.SS` (Shanghai), 0xxxxx / 3xxxxx → `.SZ` (Shenzhen) |
| United States (美股) | `us` | USD | 1–5 letters; share classes use a dash, `BRK-B` |

What is kept per market and what is shared:

- **Per market**: the watchlist, active strategy, strategies, available cash, trading-cost settings, saved filters, price alerts, journal and dividends. The user switches market from the top bar; app pages live at `/{market}/{page}`.
- **Shared by all markets**: the position-sizing settings, the plan, and the watchlist limit, which counts all markets together.

Data sources:

- **Daily bars**: Yahoo Finance for HK and US. For A-shares, Tencent first with Yahoo as backup; HK also falls back to Tencent.
- **Quotes**: Tencent for A-shares, Yahoo for everything else.
- **Fundamentals**: Yahoo, refreshed daily.
- **Earnings dates**: HKEXnews and Yahoo for HK; Nasdaq for past US dates.
- **Freshness**: pages show "Data as of …" (数据截至 …). When a feed is unavailable, the page says it is showing stored data.

## Plans: Free, Pro and trial

These are the defaults; the administrator can change them, and the pricing page (https://stock-compass.app/en/pricing) shows the current values.

| | Free | Pro |
| --- | --- | --- |
| Watchlist, all markets combined | 50 | 500 |
| Stocks scanned each day | 50 | 500 |
| Custom strategies | 3 per market | unlimited |
| Saved watchlist filters | 2 per market | unlimited |
| Backtesting | no | yes, up to 500 tickers per run |
| Signal, price-alert and planned-stop emails | no (alerts still fire in the app) | yes |
| Weekly review email | yes | yes |
| AI apps and API keys | no | yes |
| Choosing which lots a close draws from (journal breakdown only) | no, always oldest first | yes |

**Price**: Pro is USD 8 a month or USD 60 a year, through Stripe; tax may be added at checkout. Monthly and annual include the same features.

**Free plan**: never expires and needs no card. It includes the daily checklist for all three markets, the strategy builder, the journal with rules-vs-gut attribution, T+N signal scoring, relative strength, the macro strip and sector strength, concentration and sizing hints, and CSV export of trades.

**Strategy limit**: these don't count toward the free limit:
- presets;
- the copies made at sign-up;
- deleted strategies.

Imported or shared strategies do count.

**Trial**:
- 14 days of full Pro, no card, no automatic charge.
- One per account; it is refused if trials are switched off or the user already has Pro.
- It ends on the database clock, and the account returns to Free.
- Subscribing with 49 hours or more left keeps the remaining days, and the first charge falls when the trial would have ended.
- A reminder email comes in the last 3 days.

**Cancelling**: in Settings → Subscription (设置 → 订阅) through Stripe's billing page. Pro runs to the end of the paid period, and nothing more is charged. Deleting the account cancels at once, with no refund.

**After a downgrade** nothing is deleted. Anything over the free limits stays, but nothing new can be added until the user is back under the limit. Only the first 50 stocks are scanned: holdings first, then the most recently added.

**Other kinds of Pro**:
- granted by an administrator;
- lifetime Pro for early users, shown in Settings → Subscription as "Early adopter (lifetime Pro)" (早期用户永久 Pro).

## Signals and the daily scan

**Signal types**:
- `buy`, `add`, `trim` and `exit` come from strategies. The site shows them as Buy (买入), Add (加仓), Trim (减仓) and Exit (清仓).
- `price_alert` comes from the user's price alerts.
- There is no separate stop-loss signal; a stop is an exit rule.

**Timing**:
- **Buy and add** are recorded only from a **final** daily bar. A bar is final after 16:15 in HK, 15:05 for A-shares and 16:05 in New York, local time.
- During the session, the dashboard can show a buy or add on today's unfinished bar, marked "Intraday · confirmed at close" (盘中 · 收盘后确认). It counts only if the close still meets the rule. It is not logged before the close, and it cannot be shared.
- **Trim and exit** fire live during the session, at the price when the scan ran.
- Each signal is dated by the bar it was computed on, not by when the scan ran.

**Scan schedule** (weekdays):
- HK and A-shares at 09:00, 13:00, 14:00 and 15:00 Hong Kong time.
- US at 09:00, 12:00 and 15:00 New York time.
- The 09:00 run records the previous day's final buy and add signals. Runs during the session check holdings and price alerts.

**Which signals the user sees**:
- The dashboard and the API show the strategy active in each market. The scan logs every strategy's signals, so each strategy keeps its own record.
- One signal is logged per strategy, stock, type and bar date.

**Scoring**: every signal is scored at T+1, T+3, T+5 and T+10 trading days, as the close then against the signal price, adjusted for splits.

## Pages

### Today's Signals (今日信号)

The dashboard.


- **Toolbar**: the date, the data time, the active-strategy picker (changing it makes another strategy active for this market), **Share list** (分享清单) and **Refresh** (刷新数据).
- **Header above every market page**:
  - a macro strip of indices, FX, yields and VIX, depending on market;
  - Total Assets (总资产), Holdings (持仓市值), Total P&L (整体盈亏) and Available Cash (可用现金).
- **Discipline** (纪律) bar, facts only:
  - sell signals not yet acted on;
  - holdings at or below their stop;
  - holdings with no stop;
  - this month's opens by trigger.
- **Today's actions** (今日行动摘要): cash as a share of assets, today's buy and add signals, and holdings whose trim or exit rules fired.
- **Tabs**: Buy (买入), Add (加仓), Trim (减仓), Exit (清仓) and Watch (继续观察), with a sector filter. Signals appear all at once when the scan finishes.
- **Signal cards** show:
  - the price and change, matched indicators, and a chip when earnings are due within days, today or tomorrow;
  - a pre-market gap flag at 3% or more;
  - for holdings, the cost, days held and P&L.
- **Buy and add cards** also show:
  - a sizing line from the user's risk settings, such as "Your sizing allows N sh ≈ $X (P%) · stop $S" (按你的仓位规则可买 N股 ≈ $X (P%仓位) · 止损 $S). HK is rounded to board lots;
  - earnings warnings.
- **Card buttons**: **Record buy** (记录买入) and **Share** (分享). Share images never show size, cost or P&L.
- **Position risk** (持仓风险): each holding's distance to its stop ("near" is within 3%), its target and days held, with **Set stop** (设止损) or **Edit** (改) to set or change levels.
- **Concentration Risk** (持仓集中度警告): flags any stock over the single-position limit and any sector at 40% or more of assets.
- **Expected dividends** for the next 90 days.

### Watchlist (股票池)

- **Views**:
  - tabs: All (全部), Holding (持仓) and Watching (观察); Holding is the default;
  - search, cards or a compact table, and a Simple or Pro detail level;
  - sorting: RSI, today's change, biggest loss or gain, plan status, valuation discount and relative strength;
  - filters: signal, "buy conditions still met" for holdings, RSI band and plan status;
  - saved filters, called presets: **Save preset** (添加预设);
  - an earnings calendar for the next 14 days.
- **Adding stocks**: one at a time, or a pasted list for bulk import. Each ticker is checked against market data first; one with none can still be added. There is no CSV or broker-file import.
- **Cards** show:
  - cost and P&L after buy and estimated sell fees;
  - whether each strategy's buy rules are still met;
  - relative-strength rank and earnings chips;
  - lots, oldest first;
  - plan status against the stop, target and the trim/exit rules.

  The Pro detail level adds valuation and quality figures and analyst targets.
- **Detail panel**:
  - **Data summary** (数据汇总): valuation, financials, price action and events, stated as facts;
  - holdings: whether fundamentals have moved against the user since 90 days ago and since the purchase;
  - **Signal history** (信号历史): every signal with its T+N returns.
- **Removing a stock** removes it from the watchlist, with its price alerts and sector override. Trades, lots, journal, signals and cash are **kept**. A stock that is still held must be sold (closed) first.

### Strategies (策略管理)

- The list shows the active strategy first; any other can be made active with **Use this strategy** (使用此策略).
- Each card has:
  - **Backtest** (回测), **Edit** (编辑) and **Share** (分享);
  - **View rules** (查看规则), **Export JSON** (导出 JSON) and **Mute email 24h** (邮件静音 24 小时);
  - **Delete** (删除). Each market keeps at least one strategy. Deletion is soft, so past signals and journal entries keep the strategy's name.
- The card footer shows the last 30 days' buy signals and their average T+5 return. Under 10 mature signals, it says how many more are needed.
- Rules, the builder, JSON import, sharing, backtests and the live-performance page are covered in [strategies.md](strategies.md).

### Sectors (板块分析)

13 sectors, each scored 1–10 for strength. The score combines relative return, breadth (share above MA20 and MA50) and momentum, over 1 week to YTD (3 months by default), against HSI, CSI 300 or the S&P 500.

The sample is stocks on users' watchlists, not the whole market. The page also shows the user's own holdings by sector, and can classify stocks that have no sector.

Relative strength (RS) is a stock's 20-day return minus its benchmark's, ranked within the user's watchlist.

### Journal (我的复盘)

Covered under [Trade journal](#trade-journal) and [Execution discipline](#execution-discipline) below.

### Dividends (股息)

A log of dividends received, per market, with totals per currency. On a phone it is in the menu.

- **Log a dividend** (登记股息) takes the ex-date, pay date, the actual amount per share, the currency, shares and a note.
- **Add to available cash** (计入可用现金) adds the net amount received to cash; deleting the entry takes it back out.
- Dividends never change cost basis.

### Settings (设置)

| Section | What it holds |
| --- | --- |
| Account (账户) | Email, password, sign-in method, sign out |
| Subscription (订阅) | How Pro was obtained, the subscription status, the Stripe billing page |
| General (通用) | Language, default market, card detail level, theme, up/down colours (red-up or green-up) |
| Trading Cost (交易成本) | Fee schedule per market (see below) |
| Position Sizing (仓位风控) | Risk per trade, single-position limit, HK lot size |
| Notifications (邮件通知) | Signal emails (Pro) and the weekly review |
| Price Alerts (价格提醒) | The user's alerts, per market |
| AI apps & API (AI 应用与 API) | MCP address, connected apps, API keys (Pro) |
| Data & Privacy (数据与隐私) | Export, account deletion |
| Help & Docs (帮助文档), Contact (联系我们) | Guides, glossary, data sources; support@stock-compass.app |

## Recording trades: cash, costs, lots and P&L

**The trade form** has two tabs, **Buy / Add** (买入 / 加仓) and **Sell / Trim** (卖出 / 减仓).

- **Fields**: price (up to 3 decimals), whole shares, and the trade date, which defaults to today in that market's time zone.
- **Selling more** than was held on that date is refused. The earlier buy has to be recorded first.
- **The save is one transaction**: the trade, the cash movement and the recomputed holding are saved together.
- **Afterwards**: every buy opens the journal's entry form, and a sale to zero opens the close form.
- **Before you buy** (下单前看一眼) appears for a buy dated today. It never blocks. It shows:
  - whether the active strategy has a signal now or had one within 10 days;
  - whether the price is chasing: 5% or more above the previous close, or 15% or more above MA20;
  - how the user's past chasing buys did against their other buys.

**Editing and deleting trades**:
- **Edit** (改) changes price, quantity or date; cash moves by the difference.
- × deletes a trade and restores cost, quantity and cash.
- A buy whose shares were later sold cannot be deleted until the sell is changed.
- Deleting a sell does not reopen the journal rows it closed.

**Available Cash** (可用现金) is optional and per market, entered by clicking it in the header.
- A buy subtracts price × quantity + fees; a sell adds price × quantity − fees.
- If cash is left empty for a market, trades do not start tracking it.

**Fees** are kept per market, in Settings → Trading Cost (设置 → 交易成本). When a trade is recorded on the website, the fee comes from the settings in force at that moment. A trade recorded through the API can carry what the broker actually charged; that fee is kept and used instead ([api.md](api.md#cost-fees-and-pl)).

| Market | Presets | Default (moomoo) |
| --- | --- | --- |
| HK | moomoo, Futu, Tiger, IBKR, Custom | HK$15 platform fee + 0.03% commission (min HK$3) + 0.018% other fees + 0.10% stamp duty, each side |
| US | moomoo, Futu, Tiger, IBKR, Custom (commission-free) | US$0.0049/share (min $0.99) + platform $0.005/share (min $1) + 0.0023% SEC/TAF |
| A-shares | one preset | 0.025% (min ¥5) + 0.05% stamp duty on sells + 0.001% |

Every field can be edited.

**Average cost and P&L**:
- The holding's cost is the weighted average of its buys, and a sale does not change it.
- After a sale to zero, the next buy starts a new average.
- P&L is net of fees everywhere, the position-risk panel included: cost includes buy fees, and value subtracts estimated sell fees. Buy fees are estimated from the current settings, except those reported through the API, which count as they were.

**Lots** are kept first in, first out: each buy is a lot, and sales use the oldest lots first.
- On Pro, the close form can choose which lots to report as closed. That changes only the journal's per-lot breakdown; the lot records still close oldest first.
- Lots are a breakdown; cost and P&L use the average.

**Splits**:
- When the data provider re-bases a stock's price history, the user is asked to confirm the ratio: "1 share became N" (1 股变成 N 股).
- Applying it adjusts earlier trades, open journal levels and untriggered alerts. Shares are multiplied and prices divided, so total cost is unchanged.
- It is applied once.

## Trade journal

Every buy creates a journal entry, **Log open position** (记录开仓) or **Log add-on** (记录加仓), holding:

- **Trigger**: Strategy signal (策略信号), Discretionary (手动决定) or Other (其他).
  - It defaults to a signal when the stock had a matching signal in the last 14 days.
  - Back-filled holdings are marked imported; a price alert (价格提醒) is also tracked as a trigger.
- **Thesis** (optional).
- **Entry plan**:
  - The planned stop is required unless **No stop for this position** (这笔不设止损) is ticked. One-tap presets: the strategy's stop, −5%, −8% or −10%.
  - The planned target is optional.
  - The entry plan is never rewritten. Later changes go into the **current** stop and target, and every stop change is logged.
  - The position-risk panel, the alerts and the chart use the current levels. The execution record judges against the entry plan.

**Closing**, with **Close position** (平仓):
- **Reason**: Exit rule fired (触发清仓规则), Trim rule fired (触发减仓规则), Manual stop-loss (主动止损), Manual take-profit (主动止盈) or Other (其他).
- **Deviated from plan?** (是否偏离原计划) This is pre-set to yes when the price closed below the stop more than 2 trading days before the sale, and a yes needs a reason.
- **Lesson learned** (本次学到什么) is optional.
- A partial sale closes the oldest open entries first, with reason other.

**The journal page**:
- Ranges: 30, 90 or 180 days, or all (the default). **Export CSV** (导出 CSV) is on every plan.
- **Fill** (补全) back-fills holdings that have no entry.
- KPIs: total trades, closed, cumulative return (net of fees) and average holding time.
- An equity curve of realized P&L, with its maximum drawdown.
- **By rules** (按规则操作) vs **By gut feel** (凭直觉操作) compares signal trades with discretionary ones. The **Behavioral insight** (行为洞察) needs at least 5 closed trades on each side, and a gap under 1 point is called too close.
- **Attribution** (归因分析):
  - P&L by sector;
  - **Stop-Loss Discipline** (止损纪律): losing exits more than 2 points past the entry stop;
  - **Sold Too Early** (卖飞统计): winners' best close in the 10 bars after the exit;
  - holding-time buckets (0–2, 3–5, 6–10 and 11+ days);
  - results by strategy.

## Execution discipline

Everything is computed from the user's trades, signals and daily bars, never self-graded. Moves are shown without colour or verdict. Thresholds:

- a sell signal counts as acted on when the user sells within 5 calendar days;
- a stop breach (first close below the entry stop) counts as acted on when the user sells within 2 bars;
- a "chase" is a buy 5% or more above the previous close, or 15% or more above MA20;
- outcomes are measured 10 trading days later.

Cards on the journal page:

- **Execution record** (执行记录):
  - opened on a signal;
  - stop set at entry;
  - sell signals acted on within 5 days, with late and never, and the average move after the ignored ones;
  - stops acted on within 2 days;
  - chases, with their 10-day outcome against other buys.
- **Buy timing** (买入时机) sorts each buy against the active strategy's signals:
  - After a close-confirmed signal (收盘确认后买入);
  - Before the close confirmed it (盘中抢跑);
  - Before the signal (信号前买入);
  - No signal (无信号);
  - still settling, or cannot be judged.

  Each class shows its average move after 10 trading days.
- **The cost of waiting** (等待的代价) compares the signal price with the price the user actually got, times shares. It is hindsight, not a judgement.
- **Exits against the plan** (按计划复盘退出) sorts each exit:
  - held past the stop;
  - at the stop;
  - at the target (within 1%);
  - on a sell signal within 5 days;
  - unplanned.

  It also lists stop changes.

## Position risk and sizing

Position-sizing settings, in Settings → Position Sizing (设置 → 仓位风控), apply to all markets:

- **Risk per trade (% of assets)** (每笔最多亏总资产的 (%)): 0.25–3%, default 1%.
- **Max per position (%)** (单只仓位上限 (%)), the single-position limit: 5–50%, default 25%.
- **HK lot size** (港股每手股数): default 100. US rounds to 1 share and A-shares to 100.

Suggested shares are the smallest of these, rounded down to whole lots:

- the risk budget divided by the stop distance;
- the room left under the single-position limit;
- cash divided by price.

The stop distance is the active strategy's smallest `floating_loss_pct` exit threshold. On dashboard cards, the line appears only when the strategy has one; HK board lots are a reminder to check with the broker. It is a calculation from the user's own settings, not advice.

**Concentration** flags any stock above the single-position limit, as a share of total assets including cash, and any sector at 40% or more. **Got it** (知道了) hides the warning for 30 days unless the figures shift.

## Price alerts

- **Creating one**: in Settings → Price Alerts (设置 → 价格提醒), or through the API. An alert has a ticker, Above (涨破) or Below (跌破), a price and an optional note.
- **Limits**: 500 per account; duplicates are refused.
- **Plan alerts**: the planned stop and target of every holding are kept as automatic alerts, labelled Planned stop (计划止损) and Planned target (计划目标) and marked Auto (自动). They move with the plan, and the user's own alerts are never touched.
- **When one fires**:
  - It is checked at every scan: four times a day for HK and A-shares, three for the US.
  - It fires when the price is at or through the level, or when a later day's high or low crossed it.
  - It fires once, and **🔄 Re-arm** (🔄 重新启用) re-arms it.
- **Email**: fired alerts reach Pro users by email (when signal email is on), and everyone in the app.

## Emails

- **Signal emails (Pro)**:
  - The master switch is off by default.
  - After each scan, one digest goes to the sign-in email, up to 200 signals (more follow in the next one), sent between 8:00 and 17:00 market time (adjustable).
  - The user picks which types (buy/add, sell/exit, trim) and which strategies.
  - Price alerts are always included.
  - Signals older than 4 days, or from before email was switched on, are never sent.
- **Per strategy**: a bell switches email for that strategy, and **Mute email 24h** (邮件静音 24 小时) mutes it for a day. Signals are still computed and logged, and editing the strategy clears the mute.
- **Weekly review email** (每周复盘邮件), on every plan:
  - On by default, sent Monday 09:00 market time.
  - It covers the week's execution (sell signals acted on, holdings below their stop, opens on a signal and with a stop, chases, buy timing), KPIs, the best and worst trade, signal follow-through, rules vs gut over 90 days, and holdings with no stop.
  - It is skipped in a week with no opens, closes or holdings.
- **Other emails**: reminders in the first days after sign-up if nothing has been recorded, and the trial-ending notice.
- **Unsubscribing**: the link in an email turns off both the signal email and the weekly review.

## Fundamentals and the data summary

- **The Pro detail level** shows PE with its percentile among at least 8 peers in the same market and sector, PB, ROE, yield, a model estimate of fair value against the price, and analyst targets.
- **The data summary** in the detail panel states facts in four parts: valuation, financials, price action, and events and data.
  - It uses no words like cheap, strong, weak or healthy, and says when something is unknown.
  - It flags earnings within 5 days.
- **Earnings**: there is no automatic earnings blackout. Users can add `days_to_next_earnings` to their rules.

## Your data

- **Export**: Settings → Data & Privacy → **Export all my data** (设置 → 数据与隐私 → 导出我的全部数据) downloads every table as one JSON file, with row counts. That includes:
  - watchlist, trades, lots and journal;
  - strategies and their history;
  - alerts, dividends and signals;
  - settings and profile.
- **Trades only**: CSV export from the journal.
- **Deletion**: **Delete account** (删除账户) is self-service; the user types their email to confirm.
  - It cancels any subscription at once, with no refund, then deletes everything.
  - What remains: Stripe's own payment records, an address on the unsubscribe list if it is there, and one deletion record.
- **Retention**: signals and the journal are kept while the account exists.
- **Policy**: see https://stock-compass.app/en/privacy and https://stock-compass.app/en/terms.

## Public pages

- `/en` and `/zh`: the landing page.
- `/en/demo`: a read-only demo with sample data.
- `/en/pricing`: plans and the FAQ.
- `/en/guides`: 15 guides on stops, sizing, backtesting, journaling and more.
- `/en/glossary`: metric definitions.
- `/en/about`: who runs it. Stock Compass is an Australian sole trader; support is support@stock-compass.app.

Every public page has a Chinese twin under `/zh/…`. A summary for AI assistants is at https://stock-compass.app/llms.txt.

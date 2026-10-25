# Strategies, rules, indicators and backtests

Read this when the user wants to understand, write, change, import or backtest a Stock Compass strategy, or asks what an indicator or rule means.

**Labels.** The site is in English and Chinese. Pages, buttons and figures are named as the English site shows them, followed by the Chinese site's label in brackets: **Backtest** (回测). The indicator tables give both names in their own columns. When you point the user to something, use the label for the language their site is in.

## What a strategy is

A strategy is the user's own trading rules for one market (`hk`, `us` or `cn`). It has a name and four rule sections:

| Section | In the builder | Checked for | Produces |
| --- | --- | --- | --- |
| `buy` | Buy conditions (买入条件) | stocks the user does **not** hold | buy signals |
| `add` | Add conditions (加仓条件) | holdings | add signals |
| `trim` | Trim conditions (减仓条件) | holdings | trim signals |
| `exit` | Exit conditions (清仓条件) | holdings | exit signals |

How a stock is assigned to a bucket:

- A holding is checked in the order exit, then trim, then add, and gets the first section that fires. Otherwise it is "watch", shown as Watch (继续观察).
- A stock that is not held is checked only against buy. Otherwise it is "watch". A holding never gets a buy signal.

Other properties:

- Each market has one active strategy, marked **Active** (当前使用). The dashboard and `get_signals` show the active strategy's signals.
- The daily scan evaluates every one of the user's strategies in that market and logs their signals, so each strategy's own record stays complete.
- A strategy's market is fixed when it is created.

### Stops, targets, time stops and position size

A strategy has **no separate** stop-loss, take-profit, trailing-stop, time-stop or position-size settings. They are written as conditions:

| Idea | Write it as |
| --- | --- |
| Stop loss | `floating_loss_pct > 8` in `exit` (or `trim`) |
| Take profit | `floating_gain_pct > 20` in `trim` or `exit` |
| Time stop | `holding_days > 20` |
| Trailing stop (approximate) | `pct_from_high_20d < -8`, optionally AND `floating_gain_pct > 0` |

`floating_gain_pct` and `floating_loss_pct` are measured against the holding's average cost. Each has a value only when the position is up (gain) or down (loss) respectively. `holding_days` counts weekdays since the oldest open lot.

Position size comes from the user's sizing settings (Settings → Position Sizing, 设置 → 仓位风控), not from the strategy. The suggested size uses the strategy's smallest `floating_loss_pct` exit threshold as the stop distance; without one, it uses 2×ATR(14). See [product.md](product.md#position-risk-and-sizing).

A new strategy in the builder starts with:

- buy: `RSI < 40`;
- add: empty;
- trim: `floating_gain_pct > 20`;
- exit: `floating_loss_pct > 18`.

## Rule structure

Each section is a list of **groups**:

- Conditions inside a group are joined by the group's `innerOp` (AND or OR).
- The groups are joined by the section's `outerOp` (AND or OR).

```json
{ "v": 2, "outerOp": "OR",
  "groups": [
    { "innerOp": "AND", "conditions": [
      { "indicator": "rsi", "operator": "<", "value": 35 },
      { "indicator": "ma20_above_ma50", "operator": "==", "value": true }
    ] },
    { "innerOp": "AND", "conditions": [
      { "indicator": "is_20d_breakout", "operator": "==", "value": true },
      { "indicator": "vol_vs_avg20d", "operator": ">", "value": 1.5 }
    ] }
  ] }
```

That example reads: (RSI below 35 AND MA20 above MA50) OR (a 20-day breakout AND volume above 1.5× its 20-day average).

Operators depend on the indicator's type:

- **numeric**: `<`, `>`, `<=`, `>=`, `==`.
- **boolean**: `==` with `true` or `false`.
- **category**: `==`, `!=`, `IN`, `NOT IN`. `IN` and `NOT IN` take a list of strings.

How evaluation behaves:

- A condition whose indicator has **no value** for the stock is **not met**. Missing data never fires a rule.
- A missing or non-numeric threshold is not met, and an empty `IN` / `NOT IN` list is not met.
- An empty section never fires.
- An unclassified stock fails any `sector` condition.
- The "days since" indicators (`macd_golden_cross_days`, `days_above_ma20`, `trix_above_signal_days`, `vol_ma20_above_ma60_days`) are empty while their state is false. So the condition fails rather than reading zero.
- Indicators are computed from daily bars, up to about 420 days of them.

`sector` values are codes: `tech`, `semiconductor`, `ev_auto`, `healthcare`, `consumer_disc`, `consumer_staples`, `financial`, `industrials`, `energy`, `materials`, `comm_services`, `utilities`, `real_estate`. `industry` is free text from the data provider.

## Importing and exporting (JSON)

On the Strategies page (策略管理), **Export JSON** (导出 JSON) copies a strategy, and **Paste JSON to import** (粘贴 JSON 导入) creates one. An assistant can help the user write a strategy and hand them this JSON to paste:

```json
{
  "name": "Breakout with volume",
  "description": "20-day breakout on rising volume; 8% stop, trim at +20%, time stop at 30 days.",
  "market": "us",
  "rules": {
    "buy":  { "v": 2, "outerOp": "AND", "groups": [ { "innerOp": "AND", "conditions": [
      { "indicator": "is_20d_breakout", "operator": "==", "value": true },
      { "indicator": "vol_vs_avg20d", "operator": ">", "value": 1.5 },
      { "indicator": "ma20_above_ma50", "operator": "==", "value": true }
    ] } ] },
    "add":  { "v": 2, "outerOp": "AND", "groups": [] },
    "trim": { "v": 2, "outerOp": "OR",  "groups": [ { "innerOp": "AND", "conditions": [
      { "indicator": "floating_gain_pct", "operator": ">", "value": 20 }
    ] } ] },
    "exit": { "v": 2, "outerOp": "OR",  "groups": [
      { "innerOp": "AND", "conditions": [ { "indicator": "floating_loss_pct", "operator": ">", "value": 8 } ] },
      { "innerOp": "AND", "conditions": [ { "indicator": "holding_days", "operator": ">", "value": 30 } ] }
    ] }
  }
}
```

Import rules:

- `name` is required, up to 60 characters.
- `market` is `hk`, `us` or `cn`. If it is left out, the current market is used.
- `buy` must have at least one condition. The builder requires this too: a strategy without one can never give a buy signal.
- Every indicator key must exist; a near miss gets a "did you mean" suggestion.
- Operators must suit the indicator's type. Boolean values may be `true` / `false` or `1` / `0`.
- After import, the builder opens so the user can review and save.

Things to know when writing one:

- Imported strategies count toward the free plan's limit of 3 custom strategies per market.
- After a rule change, the strategy's live statistics restart from the change. Its history is kept.
- There is no API or MCP tool for creating or changing strategies. The user pastes the JSON on the Strategies page and saves it there.

## Sharing

**Share** (分享) creates a new link `https://stock-compass.app/s/<code>` with a snapshot of the rules only. It never includes holdings, trades or results.

- Anyone can view the link.
- Taking the strategy signs the visitor up if needed and adds it to their strategies. Pro users go straight to a backtest of it.
- The app has no way to withdraw a link once it is shared.

## Backtest (Pro)

Run from a strategy's **Backtest** (回测) button, on the strategy's market watchlist. The universe is **Full watchlist** (全部股票池) or **Holdings** (仅持仓), up to 500 tickers taken in symbol order.

How trades are simulated:

- **Data**: the stored daily bars, about 2 years for HK and US and about 1 year for China A-shares. A ticker needs at least 30 bars.
- **Fills**: a signal on day D's close fills at day D+1's **open**. Stops are checked on closes and filled at the next open, so overnight and earnings gaps go straight through a stop, as they do in real trading.
- **Costs**: fees from the user's trading-cost settings, charged on the actual order size. No slippage is modelled.
- **Capital**: **Capital** (总资金), default 100,000, split across **Positions** (目标持仓数) slots, default 5, up to 50.
  - Sizing is **Equal** (等额) or **Risk-sized** (风险仓位); risk-sized needs a `floating_loss_pct` exit rule.
  - When every slot is full, new buy signals are skipped and counted.
  - Same-day buys fill in ticker order.
- **Not simulated**:
  - **add rules** — a position is opened once;
  - **repeated trims** — a trim sells half the position, once per position.
- **Indicators it cannot evaluate** are removed and the run is marked **PARTIAL** (部分回测). These include:
  - every valuation and fundamental indicator;
  - `macd_signal`, `kdj_k` and `kdj_d`;
  - `turnover_rate`, `beta_index_20d`;
  - `sector` and `industry`.

  `days_to_next_earnings` works only where the earnings calendar covers the date. The "Backtest" column in the table below shows each indicator.

What it reports:

- Final equity (期末资金), Cum. net (累计净收益), the cumulative net return, and Annualized (年化);
- Max DD (最大回撤), the maximum drawdown, Profit factor (盈亏比) and Win rate (胜率);
- Avg return (平均收益), plus a money-weighted version;
- Trades (交易次数), with skipped signals;
- Avg days held (平均持仓(交易日)) and Avg invested (平均仓位占用), the average capital in use.

Results can be shown gross or net of costs; net is the default. Two diagnostics come with them:

- **Overfitting check**: a 70/30 in-sample vs out-of-sample split by date flags "Out-of-sample degraded materially — likely overfit" (样本外表现明显变差 — 可能过拟合) when the win rate falls more than 15 points or the annualized return turns from positive to negative. Each segment needs at least 10 closed trades.
- **Buy-rule funnel**: shows which condition removed the candidates.

Past results do not predict future ones. Report backtest numbers as what the rules would have done on the data, nothing more.

## Live Performance (实盘表现)

Each strategy has a live-performance page (`/{market}/strategy/{id}/effectiveness`) built from the signals it actually recorded:

- For **buy signals**: the win rate and average return at T+1, T+3, T+5 and T+10 trading days after the signal bar, with how many are mature.
- A cumulative curve ($1 per buy signal, held to T+10).
- A table of signals.

Ranges are 30, 90 or 180 days, or all; 30 is the default. Returns are gross by default, and the net view subtracts a flat round-trip cost. After a rule change, it counts signals since the change unless **Include earlier signals** (包含修改前) is ticked.

## Indicators

<!-- 96 indicators, generated from src/lib/metricsConfig.ts -->
### Valuation (估值)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `pe` | numeric | PE | PE (市盈率) | P/E ratio. Common reading: <15 low, >30 high; varies widely by sector | no |
| `pb` | numeric | PB | PB (市净率) | P/B ratio. <1 below book value | no |
| `dividend_yield` | numeric | Dividend Yield (%) | 股息率 (%) | Annual dividend / price. >4% = high-yield | no |
| `peg_ratio` | numeric | PEG | PEG | PE / earnings growth. <1 reasonable, >2 expensive | no |
| `forward_pe` | numeric | Forward PE | Forward PE | PE based on forward 1y earnings | no |
| `market_cap_hkd_100m` | numeric | Market Cap (100M) | 市值 (亿) | Market cap in 100 million (local currency) | no |

### Trend & Momentum (趋势 & 动量)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `rsi` | numeric | RSI(14) | RSI(14) | RSI 0-100. <35 oversold, >70 overbought | yes |
| `macd_dif` | numeric | MACD DIF (fast line) | MACD DIF (快线) | DIF = EMA12 - EMA26, the MACD line. DIF > 0 means above the zero axis. Combine with MACD golden-cross days to express a cross above zero: cross days <= 3 AND DIF > 0. | yes |
| `macd_bullish_divergence` | boolean | MACD bullish divergence | MACD 底背离 | Price made a LOWER low, but DIF at this golden cross sits at least 15% higher than at the previous one — momentum deteriorating less than price. Only crosses below the zero axis count. Fewer than two such crosses reads as no data, not as no divergence. | yes |
| `macd_hist` | numeric | MACD Histogram (absolute) | MACD柱状图 (绝对值) | DIF - DEA. ⚠️ Only the SIGN is comparable across stocks: the value is in price units, so the same pattern reads ~0.09 on a $12 stock and ~8.7 on a $1200 one — a 100x spread. "> 0 / < 0" is meaningful; comparing against a number like 40 is not. Use the %-of-price version for magnitude. | yes |
| `macd_hist_pct` | numeric | MACD Histogram (% of price) | MACD柱状图 (占股价%) | Histogram / price x 100. Comparable across stocks — use it to say "momentum is strong enough" rather than only "direction is positive". e.g. > 0.2 means the histogram is 0.2% of the share price. | yes |
| `macd_golden_cross_days` | numeric | Days since golden cross | MACD金叉天数 | Days since the last golden cross. 0=today, <=3 is fresh. No cross in the window now reads as no data (condition unmet); it no longer returns -1, which used to satisfy -1<=3 and let never-crossed stocks pass a fresh-cross filter. For "currently in a dead cross" use MACD histogram < 0. | yes |
| `macd_signal` | numeric | DEA (signal line) | DEA (慢线/信号线) | DEA = the 9-day EMA of DIF, i.e. the MACD signal line itself — NOT the DIF-minus-DEA difference, which is macd_hist. In price units, so only the sign is comparable across stocks: use > 0 to mean "DEA above the zero axis", not a numeric threshold. | no |
| `adx_14` | numeric | ADX(14) | ADX(14) | Trend strength 0-100. >25 strong, <20 choppy | yes |
| `ma5_above_ma20` | boolean | MA5 > MA20 | MA5 > MA20 | 5d MA above 20d MA = short-term uptrend | yes |
| `ma10_above_ma20` | boolean | MA10 > MA20 | MA10 > MA20 | 10d MA above 20d MA = short/mid-term MAs aligned bullish | yes |
| `ma20_above_ma30` | boolean | MA20 > MA30 | MA20 > MA30 | 20d MA above 30d MA = mid-term trend up | yes |
| `days_above_ma20` | numeric | Days above MA20 | 站上MA20天数 | Consecutive bars, counting back from the latest, that closed above the 20d MA. 1 = crossed above today, 4 = has held above for four bars. Note: when the latest close is NOT above MA20 this reads as no data rather than 0 — a 0 would let "days above <= 4" pass for a stock that never crossed, while null makes every comparison fail closed. For "below MA20" use Price vs MA20 < 0. | yes |
| `ma20_above_ma50` | boolean | MA20 > MA50 | MA20 > MA50 | 20d MA above 50d MA = mid-term uptrend | yes |
| `ma20_slope_5d_pct` | numeric | MA20 5d slope (%) | MA20 5日斜率 (%) | Slope of 20d MA over last 5 days | yes |
| `price_vs_ma20` | numeric | Price vs MA20 (%) | 价格vs MA20 (%) | Price deviation from 20d MA | yes |
| `price_vs_ma50` | numeric | Price vs MA50 (%) | 价格vs MA50 (%) | Price deviation from 50d MA | yes |
| `price_vs_ma200` | numeric | Price vs MA200 (%) | 价格vs MA200 (%) | Price deviation from 200d MA. Long-term trend | yes |
| `return_5d` | numeric | 5d return (%) | 5日涨跌幅 (%) | 5-day cumulative return | yes |
| `return_10d` | numeric | 10d return (%) | 10日涨跌幅 (%) | 10-day cumulative return | yes |
| `return_20d` | numeric | 20d return (%) | 20日涨跌幅 (%) | 20-day cumulative return | yes |
| `pct_from_high_10d` | numeric | Dist from 10d high (%) | 距10日高点 (%) | Negative = pullback depth from 10d high | yes |
| `pct_from_low_10d` | numeric | Dist from 10d low (%) | 距10日低点 (%) | Positive = bounce above 10d low | yes |
| `pct_from_high_20d` | numeric | Dist from 20d high (%) | 距20日高点 (%) | Distance from 20d high | yes |
| `pct_from_low_20d` | numeric | Dist from 20d low (%) | 距20日低点 (%) | Distance from 20d low | yes |
| `pct_from_high_60d` | numeric | Dist from 60d high (%) | 距60日高点 (%) | Distance from 60d high | yes |
| `pct_from_low_60d` | numeric | Dist from 60d low (%) | 距60日低点 (%) | Distance from 60d low | yes |
| `dist_from_52w_high_pct` | numeric | Dist from 52w High (%) | 距52周高点 (%) | % drop from 52w high | yes |
| `dist_from_52w_low_pct` | numeric | Dist from 52w Low (%) | 距52周低点 (%) | % above 52w low | yes |
| `consec_up_days` | numeric | Consec up days | 连续上涨天数 | Current streak of up-close days | yes |
| `consec_down_days` | numeric | Consec down days | 连续下跌天数 | Streak of down days. >=3 oversold confirm | yes |
| `trix` | numeric | TRIX(12) | TRIX(12) | Rate of change of a triple-smoothed EMA, in percent. The triple smoothing is the point — it strips short cycles, so the sign reads as a slow trend rather than a daily move. Percent units, so it is comparable across stocks (unlike DIF). > 0 is the zero-axis test. | yes |
| `trix_above_signal_days` | numeric | Days TRIX > signal | TRIX金叉天数 | How many bars TRIX has been above its signal line TRMA = MA(TRIX,20). 1 = crossed today. Uses the TRIX(12,20) parameters standard on CN/HK charting software, not the 9-period EMA some US platforms use. Reads as no data (condition unmet) while TRIX is below the signal. Pair with TRIX > 0 for "golden cross above the zero axis". | yes |

### Volume & Money Flow (成交量 & 资金流)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `vol_vs_avg5d` | numeric | Vol vs 5d Avg | 成交量vs5日均量 (x) | Today's vol / 5d avg | yes |
| `vol_vs_avg10d` | numeric | Vol vs 10d Avg | 成交量vs10日均量 (x) | Today's vol / 10d avg | yes |
| `vol_vs_prev` | numeric | Volume vs previous bar (x) | 量比前一日 (倍) | Today's volume / the previous bar's volume. Unlike vol vs 5/10/20d averages, this compares a single prior bar — use it to catch a sudden volume spike. 2 = double. For a range (2x to 8x) AND two conditions. Zero or missing previous volume reads as no data. | yes |
| `vol_vs_avg20d` | numeric | Vol vs 20d Avg | 成交量vs20日均量 (x) | Today's vol / 20d avg. 1.5x = surge | yes |
| `turnover_rate` | numeric | Turnover (%) | 换手率 (%) | Volume / float × 100. >3% active | no |
| `vol_ma20_above_ma60_days` | numeric | Days vol MA20 > MA60 | 量能金叉天数 (量MA20>MA60) | How many bars the 20-day average volume has been above the 60-day average. 1 = crossed today, <=3 is a fresh cross, larger means the expansion has held longer. Reads as no data (condition unmet) while vol MA20 is still below MA60 — not 0, which would let never-crossed stocks pass a "<=3" filter. For a single-bar volume spike use Volume vs previous bar. | yes |
| `obv_slope_5d` | numeric | OBV 5d slope | OBV 5日斜率 | 5d OBV change %. >0 = inflow | yes |
| `mfi_14` | numeric | MFI(14) | MFI(14) | Money Flow Index 0-100. <20 oversold, >80 overbought | yes |

### Volatility (波动率)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `atr14` | numeric | ATR(14) | ATR(14) | 14-day Average True Range (price units) | yes |
| `atr14_pct` | numeric | ATR(14) % | ATR(14) % | ATR as % of price. >3% high volatility | yes |
| `atr_change_pct` | numeric | ATR change (%) | ATR变化率 (%) | ATR vs 20d avg ATR. >0 expanding vol | yes |
| `realized_vol_20d_pct` | numeric | 20d Realized Vol (%) | 20日年化波动率 (%) | 20d return std × √252 | yes |
| `price_vs_bb_lower` | numeric | BB Lower (%) | 布林带位置 (%) | Distance from BB lower band | yes |
| `bb_width_pct` | numeric | BB Width (%) | 布林带带宽 (%) | Low width = squeeze, possible breakout | yes |
| `today_change_pct` | numeric | Today's Change (%) | 今日涨跌幅 (%) | Today's price change % | yes |
| `daily_range_pct` | numeric | Daily range (%) | 今日振幅 (%) | (high-low)/prev close × 100 | yes |
| `gap_pct` | numeric | Gap (%) | 跳空幅度 (%) | (open-prev_close)/prev × 100. >0 = gap up | yes |
| `body_to_range_pct` | numeric | Body/range (%) | K线实体占比 (%) | \|close-open\|/(high-low)×100. Large = strong day | yes |

### Candlestick (K线信号)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `is_fake_bearish` | boolean | Fake bearish candle | 假阴真涨 | Close below open (a red candle) but close above the previous close — down against its own open, up against yesterday. Often a gap-up that faded yet still finished higher. Insufficient bars or a missing open reads as no data. | yes |
| `kdj_k` | numeric | KDJ K | KDJ-K值 | KDJ K. <20 oversold, >80 overbought | no |
| `kdj_d` | numeric | KDJ D | KDJ-D值 | KDJ D. <20 oversold, >80 overbought | no |

### Candle Patterns (K线形态)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `is_hammer` | boolean | Hammer | 锤子线 | Long lower wick (≥2× body), short upper wick, after a downtrend. Bullish reversal. | yes |
| `is_bullish_engulfing` | boolean | Bullish Engulfing | 看涨吞没 | Today's up-body fully engulfs yesterday's down-body. Bullish reversal. | yes |
| `is_bearish_engulfing` | boolean | Bearish Engulfing | 看跌吞没 | Today's down-body engulfs yesterday's up-body. Bearish reversal. | yes |
| `is_doji` | boolean | Doji | 十字星 | Tiny body (<10% of range) — indecision, often precedes reversal. | yes |

### Structure (趋势结构)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `is_20d_breakout` | boolean | 20d Breakout | 突破20日高 | Close breaks above the prior 20-day high. Trend breakout. | yes |
| `is_20d_breakdown` | boolean | 20d Breakdown | 跌破20日低 | Close breaks below the prior 20-day low. Trend breakdown. | yes |
| `days_since_swing_high` | numeric | Days since swing high | 距上个波段高点(天) | Trading days since the last swing high. Small = just peaked. | yes |
| `days_since_swing_low` | numeric | Days since swing low | 距上个波段低点(天) | Trading days since the last swing low. Small = just bottomed. | yes |

### Fundamentals (基本面)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `roe_pct` | numeric | ROE (%) | ROE (%) | Return on Equity. >15% quality | no |
| `roa_pct` | numeric | ROA (%) | ROA (%) | Return on Assets | no |
| `revenue_growth_yoy_pct` | numeric | Revenue Growth YoY (%) | 营收增长率 YoY (%) | Year-over-year revenue growth | no |
| `earnings_growth_yoy_pct` | numeric | Earnings Growth YoY (%) | 净利增长率 YoY (%) | YoY earnings growth | no |
| `gross_margin_pct` | numeric | Gross Margin (%) | 毛利率 (%) | Gross margin = moat indicator | no |
| `profit_margin_pct` | numeric | Profit Margin (%) | 净利率 (%) | Net profit margin | no |
| `debt_to_equity` | numeric | Debt/Equity | 负债率 | Lower = safer | no |
| `days_to_next_earnings` | numeric | Days to next earnings | 距下次财报天数 | Days until next earnings. Prices can gap around earnings, straight past a stop. | partly (earnings calendar) |
| `pe_ttm` | numeric | P/E TTM | 市盈率 TTM | Price / TTM EPS | no |
| `ps` | numeric | P/S | 市销率 | Price / Sales | no |
| `pe_industry_pctile` | numeric | P/E sector %ile | PE 行业百分位 | P/E percentile among the same-market, same-sector stocks this app tracks (users' watchlists, not the whole market). No value when there are fewer than 8 peers, so the rule does not fire | no |
| `pb_industry_pctile` | numeric | P/B sector %ile | PB 行业百分位 | P/B percentile among the same-market, same-sector stocks this app tracks. No value with fewer than 8 peers | no |
| `fcf_yield_pct` | numeric | FCF Yield (%) | 自由现金流收益率 (%) | Free Cash Flow / Market Cap | no |
| `dividend_yield_pct` | numeric | Dividend Yield % (fund.) | 股息率 % (基本面) | Annual dividend / price, from fundamentals cache | no |
| `debt_to_equity_pct` | numeric | Debt/Equity % | 债务权益比 (D/E) % | Debt to equity ratio (Yahoo definition) | no |
| `liability_to_asset_pct` | numeric | Liability/Asset % | 资产负债率 % | Total liabilities / total assets — traditional Asset-Liability Ratio | no |
| `margin_of_safety_pct` | numeric | Deviation from fair value % (negative = cheap) | 偏离公允价 % (负=便宜) | (price - fair_value) / fair_value x 100. NEGATIVE = cheap, which is the opposite sign to the usual "margin of safety" convention — writing "> 20" gives you 20% OVERvalued, not a 20% margin. Use < -10 for undervalued. | no |
| `analyst_target_price` | numeric | Analyst Target | 分析师目标价 | Mean analyst price target (Yahoo consensus) | no |

### Market & Sector (大盘 & 板块)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `market_index_rsi_14` | numeric | Market RSI(14) | 大盘 RSI(14) | Benchmark RSI. <30 oversold. Benchmark auto by market (HK=HSI, CN=CSI300, US=S&P500) | yes (benchmark) |
| `market_index_vs_ma20_pct` | numeric | Market vs MA20 (%) | 大盘距 MA20 (%) | Benchmark deviation from 20d MA. Auto by market | yes (benchmark) |
| `market_index_vs_ma50_pct` | numeric | Market vs MA50 (%) | 大盘距 MA50 (%) | Benchmark vs 50d MA. Auto by market | yes (benchmark) |
| `market_index_return_5d` | numeric | Market 5d return (%) | 大盘 5日涨跌 (%) | Benchmark 5-day return. Auto by market | yes (benchmark) |
| `market_tech_vs_ma20_pct` | numeric | Tech Index vs MA20 (%) | 科技指数距 MA20 (%) | Tech subindex vs 20d MA (HK=HSTECH, CN=ChiNext, US=NDX) | yes (benchmark) |
| `beta_index_20d` | numeric | Beta vs Market 20d | 与大盘 Beta(20日) | Beta vs benchmark. >1 = more volatile. Benchmark auto by market | no |

### Classification (分类)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `sector` | category | Sector | 板块 | Stock sector | no |
| `industry` | category | Industry | 行业 | Industry (free text) | no |

### Position (仓位)

| Key | Type | English | Chinese | What it measures | Backtest |
| --- | --- | --- | --- | --- | --- |
| `floating_gain_pct` | numeric | Floating Gain (%) | 浮盈 (%) | Unrealized gain %. Holdings only | yes |
| `floating_loss_pct` | numeric | Floating Loss (%) | 浮亏 (%) | Unrealized loss %. Holdings only | yes |
| `holding_days` | numeric | Holding days | 持仓天数 | Trading days since the oldest open lot. Weekends are excluded; exchange holidays are not, so it runs about a day fast every five weeks. Only computed for holdings — the condition fails for anything you do not hold. | yes |

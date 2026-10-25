<p align="center"><img src="assets/icon.png" width="96" alt="Stock Compass"></p>

# Stock Compass for Claude

Use your [Stock Compass](https://stock-compass.app) records in a conversation with Claude. You can ask about your watchlist, holdings, signals and trades, and tell Claude to record a trade, change a stop or target, or set a price alert. It covers Hong Kong, US and China A-shares.

The plugin connects Claude to your own Stock Compass account. It also includes a skill that teaches Claude the rules:

- tickers and markets;
- what each signal means;
- confirming every change with you before it is made.

**It only keeps records.** Recording a trade writes it into your Stock Compass journal. It never places an order with a broker.

## What you need

- A Stock Compass account on **Pro**.
- Claude on the web, the desktop app, Cowork or Claude Code.

## Install

### claude.ai, the desktop app and Cowork

1. Go to **Customize → Plugins → Add → Add marketplace** and enter `Stock-Compass/stock-compass-plugin`.
2. Install **Stock Compass**.
3. On the plugin's **Connectors** tab, connect it. Stock Compass opens so you can sign in and approve.

### Claude Code

```bash
claude plugin marketplace add Stock-Compass/stock-compass-plugin
claude plugin install stock-compass@stock-compass
```

Then run `/mcp` in a session and sign in to `stock-compass`.

## Signing in and permissions

The connector signs you in with OAuth on stock-compass.app; you never copy a password or key into Claude. On the approval page you choose:

- **Read only** (the default): your watchlist, holdings, trades, signals and price alerts.
- **Read and make changes**: also add or remove watchlist stocks, record trades, change stops and targets, and set price alerts.

You can disconnect at any time under **Settings → AI apps & API** on stock-compass.app. Disconnecting stops the connection at once.

## Things to ask

- "What's on my Hong Kong watchlist, and what are my stops?"
- "Which buy signals did my strategies record this week?"
- "I bought 200 shares of 700 at 382.4 today. Record it, stop at 350."
- "Set an alert when AAPL goes below 210."

## Data

When you use it, Claude reads the records it asks for from your account through `https://stock-compass.app/api/mcp`:

- watchlist and holdings;
- trades;
- signals;
- price alerts.

It sends Stock Compass the changes you approve. What Claude reads becomes part of your conversation with Claude, and Anthropic's privacy terms cover that. Stock Compass stores your sign-in only as a hashed token, and logs each change made through the connection, which you can see in Settings. See the [privacy policy](https://stock-compass.app/en/privacy).

## Other apps

The same connector works with any app that supports remote MCP servers with OAuth, such as ChatGPT. Add `https://stock-compass.app/api/mcp` as a custom MCP server and choose OAuth.

## Support

Email support@stock-compass.app or [open an issue](https://github.com/Stock-Compass/stock-compass-plugin/issues).

---

## 中文说明

在和 Claude 的对话里使用你的 [Stock Compass](https://stock-compass.app) 记录，支持港股、美股和 A 股。能查看自选股、持仓、信号和交易，也能让 Claude 记录买卖、改止损和目标、设价格提醒。

**只记账，不会去券商下单。** 需要 Stock Compass **Pro** 账号。

**安装**

- **claude.ai 或桌面版**：
  1. 打开 Customize → Plugins → Add → Add marketplace，填 `Stock-Compass/stock-compass-plugin`。
  2. 安装 Stock Compass。
  3. 在插件的 Connectors 标签里点连接，登录 Stock Compass 并授权。
- **Claude Code**：运行上面的两条命令，再在会话里输入 `/mcp` 登录。

**授权**：默认只读。勾选「修改」后，才能改自选股、记录买卖、改止损和目标、设价格提醒。随时可以在网站「设置 → AI 应用与 API」里断开，断开后立即失效。

**数据**：Claude 只读取它需要的记录，并只提交你确认过的改动。你的登录凭证在 Stock Compass 只存哈希。每次改动都有记录，可以在设置页查看。详见[隐私政策](https://stock-compass.app/zh/privacy)。

## License

[MIT](LICENSE)

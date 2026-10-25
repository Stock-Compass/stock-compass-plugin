<p align="center"><img src="assets/icon.png" width="96" alt="Stock Compass"></p>

<p align="center"><b>English</b> · <a href="README.zh-CN.md">简体中文</a></p>

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

## License

[MIT](LICENSE)

<p align="center"><img src="assets/icon.png" width="96" alt="Stock Compass"></p>

<p align="center"><a href="README.md">English</a> · <b>简体中文</b></p>

# Stock Compass Claude 插件

在和 Claude 的对话里使用你的 [Stock Compass](https://stock-compass.app) 记录。你可以问它自选股、持仓、信号和交易，也可以让它帮你记录买卖、改止损和目标、设价格提醒。支持港股、美股和 A 股。

这个插件把 Claude 连到你自己的 Stock Compass 账户，并附带一个 skill，告诉 Claude 这些规则：

- 股票代码和市场怎么写；
- 每种信号是什么意思；
- 每次改动前都先跟你确认。

**只记账。** 记录一笔交易，就是把它写进你在 Stock Compass 的交易日志，不会去券商下单。

## 需要什么

- 一个 Stock Compass **Pro** 账号。
- Claude 网页版、桌面版、Cowork 或 Claude Code。

## 安装

### claude.ai、桌面版和 Cowork

1. 打开 **Customize → Plugins → Add → Add marketplace**，填入 `Stock-Compass/stock-compass-plugin`。
2. 安装 **Stock Compass**。
3. 在插件的 **Connectors** 标签里点连接。Stock Compass 会打开，让你登录并授权。

### Claude Code

```bash
claude plugin marketplace add Stock-Compass/stock-compass-plugin
claude plugin install stock-compass@stock-compass
```

然后在会话里输入 `/mcp`，登录 `stock-compass`。

## 登录与权限

连接器在 stock-compass.app 上用 OAuth 登录，不需要把密码或密钥复制给 Claude。在授权页上你可以选：

- **只读**（默认）：自选股、持仓、交易、信号和价格提醒。
- **读取和修改**：另外还能加入或移出自选股、记录买卖、改止损和目标、设价格提醒。

随时可以在 stock-compass.app 的 **设置 → AI 应用与 API** 里断开，断开后立即失效。

## 可以这样问

- 「我的港股自选股有哪些？止损都设在哪？」
- 「这周我的策略记录了哪些买入信号？」
- 「我今天以 382.4 买了 200 股 700，帮我记一下，止损 350。」
- 「AAPL 跌破 210 时提醒我。」

## 数据

使用时，Claude 通过 `https://stock-compass.app/api/mcp` 从你的账户读取它需要的记录：

- 自选股和持仓；
- 交易；
- 信号；
- 价格提醒。

它只会把你确认过的改动发给 Stock Compass。Claude 读到的内容会成为你和 Claude 对话的一部分，适用 Anthropic 的隐私条款。Stock Compass 只以哈希形式保存你的登录凭证，并记录每一次通过连接做的改动，你可以在设置页查看。详见[隐私政策](https://stock-compass.app/zh/privacy)。

## 其它应用

同一个连接器也能用在其它支持远程 MCP 和 OAuth 的应用里，比如 ChatGPT：把 `https://stock-compass.app/api/mcp` 添加为自定义 MCP 服务器，认证方式选 OAuth。

## 支持

发邮件到 support@stock-compass.app，或者[提交 issue](https://github.com/Stock-Compass/stock-compass-plugin/issues)。

## 许可证

[MIT](LICENSE)

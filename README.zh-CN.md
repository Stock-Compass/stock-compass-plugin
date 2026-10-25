<p align="center"><img src="assets/icon.png" width="96" alt="Stock Compass"></p>

<p align="center"><a href="README.md">English</a> · <b>简体中文</b></p>

# 在 AI 应用里使用 Stock Compass

在任何 AI 应用或 agent 里使用你的 [Stock Compass](https://stock-compass.app) 记录，支持港股、美股和 A 股。你可以：

- 问它你的自选股、持仓、信号、交易和交易日志；
- 让它帮你记录买卖、改止损和目标、设价格提醒；
- 请它帮你写策略、解读回测。

**只记账。** 记录一笔交易，就是把它写进你在 Stock Compass 的交易日志，不会去券商下单。连接需要 Stock Compass **Pro** 账号。

## 仓库里有什么

| | 作用 | 适用于 |
| --- | --- | --- |
| **MCP 连接器** `https://stock-compass.app/api/mcp` | 给 AI 9 个操作你自己记录的工具（4 个读取，5 个修改） | 任何支持远程 MCP 的应用：Claude、ChatGPT、Claude Code、Cursor 等 |
| **Skill** `skills/stock-compass/` | 让 AI 了解 Stock Compass 的全部功能和使用规则：信号、策略和全部指标、回测、交易、手续费、交易日志、执行纪律、仓位、提醒、邮件、套餐、API 密钥和 REST 接口 | 支持 Agent Skills（`SKILL.md`）的应用。其它应用可以把这些文件当普通文档读。 |
| **Claude 插件** `.claude-plugin/` | 一步装好连接器和 skill | Claude（网页版、桌面版、Cowork）和 Claude Code |

Skill 由 `SKILL.md` 和三份参考文件组成。AI 只在问题需要时才读对应的文件：

- [`SKILL.md`](skills/stock-compass/SKILL.md)：什么时候用哪个工具，以及使用规则（改动前先确认、不给买卖建议）；
- [`product.md`](skills/stock-compass/references/product.md)：每个页面、套餐、邮件和设置；
- [`strategies.md`](skills/stock-compass/references/strategies.md)：规则写法、全部 96 个指标、策略 JSON 导入、回测；
- [`api.md`](skills/stock-compass/references/api.md)：OAuth、API 密钥、MCP 工具、REST 接口和错误码。

参考文件是英文写的，AI 会用你的语言回答。

## 连接你的 AI 应用

| 应用 | 怎么连接 |
| --- | --- |
| **Claude**（网页版、桌面版、Cowork） | Customize → Plugins → Add → Add marketplace → `Stock-Compass/stock-compass-plugin`，安装 **Stock Compass**，再在插件的 **Connectors** 标签里点连接。 |
| **Claude Code** | `claude plugin install stock-compass --marketplace Stock-Compass/stock-compass-plugin`，然后运行 `/mcp` 登录。 |
| **ChatGPT** | 打开 [chatgpt.com/plugins](https://chatgpt.com/plugins) → + → Add custom MCP server，粘贴 `https://stock-compass.app/api/mcp`，认证选 OAuth。 |
| **其它 MCP 客户端** | 把 `https://stock-compass.app/api/mcp` 添加为远程 MCP 服务器（Streamable HTTP）。认证选 OAuth，或者带请求头 `Authorization: Bearer sc_…` 用 API 密钥。 |
| **脚本，或不支持 MCP 的工具** | 用 API 密钥调用 REST 接口 `https://stock-compass.app/api/v1`，见 [api.md](skills/stock-compass/references/api.md)。 |

每个应用都会把你带到 stock-compass.app 登录并确认授权：

- **只读**（默认）：自选股、持仓、交易、信号和价格提醒。
- **读取和修改**：另外还能加入或移出自选股、记录买卖、改止损和目标、设价格提醒。

随时可以在 stock-compass.app 的 **设置 → AI 应用与 API** 里断开连接或吊销 API 密钥。

## 把 skill 加到你的 AI 里

- **Claude**：装上面的插件就带了。
- **支持 Agent Skills 的应用**：把 `skills/stock-compass` 文件夹复制到该应用的 skills 目录，具体位置见各应用的文档。
- **其它 AI**：直接把上面几份文件给它，或者让它读 [stock-compass.app/llms.txt](https://stock-compass.app/llms.txt) 上的简介。

## 可以这样问

- 「我的港股自选股有哪些？止损都设在哪？」
- 「这周我的策略记录了哪些买入信号？」
- 「我今天以 382.4 买了 200 股 700，帮我记一下，止损 350。」
- 「帮我写一个突破策略，8% 止损，然后给我导入用的 JSON。」
- 「为什么我的回测显示『部分回测』？」
- 「复盘页上的『等待的代价』是什么意思？」

## 数据

使用时，AI 只读取它需要的记录，并只把你确认过的改动发给 Stock Compass。它读到的内容会成为你和这个 AI 对话的一部分，适用其提供方的隐私条款。

Stock Compass 只以哈希形式保存登录令牌和 API 密钥，并记录每一次通过连接做的改动，你可以在设置页查看。详见[隐私政策](https://stock-compass.app/zh/privacy)。

## 支持

发邮件到 support@stock-compass.app，或者[提交 issue](https://github.com/Stock-Compass/stock-compass-plugin/issues)。

## 许可证

[MIT](LICENSE)

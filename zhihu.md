# 知乎投稿稿

**标题（选一）：**
1. 我给 openEuler 做了个插件，顺手把六个 Agent 平台的「上架通道」都摸了一遍（推荐）
2. 给 CLI 加 Agent 插件：从 MCP 开发到上架的完整路线，含各平台实测结论
3. Agent 插件开发与上架全景：Claude Code、Codex、Trae、扣子……哪条通道真能走通

**话题标签建议：** 人工智能 / Agent / MCP / 开源 / openEuler / 大模型

**发布方式：** 正文从下一行开始，整体复制进知乎文章编辑器即可（表格和代码块会自动转换；个别表格粘贴错位时用编辑器自带表格工具重建）。

---

先说一个真实场景。

用户在 Trae 里问：「查一下 CVE-2019-10082 在 openEuler 的安全公告编号和严重程度」。我在 openEuler 的命令行工具 oed 上接了 MCP server，工具就在那里，装也装好了。但那一次模型没有调用任何工具，直接编了个答案——CVE 编号格式看着挺像回事，内容全是错的。

这件事逼着我把「怎么让 agent 用上你的工具、用上之后怎么让更多人装上」这条链路从头到尾走了一遍：做了 MCP server，做了 skill，做了原生插件，发了 npm，然后把 Claude Code、Cursor、Trae、Codex、Cline、DeepSeek dsh、Gemini CLI、Qwen Code 的接入方式全部实测，又把每家「官方上架渠道」逐个核实——能核实的核实，核实不了的明说，宁缺毋滥。

这篇文章就是那份完整记录：**怎么开发一款 agent 插件，以及做完之后往哪上架**。官方渠道是什么，非官方渠道是什么，以及哪些渠道看起来存在、其实走不通。

## 一、插件体系是三层，不是一个东西

很多人说「做个插件」的时候，其实混了三件事：

- **指令层**：教模型「什么时候用、怎么用」——SKILL.md、rules、系统提示词；
- **工具层**：让模型「真的能调」——MCP server 或原生插件；
- **分发层**：让用户「装得上」——包仓库、市场目录、客户端配置。

结论先行：**MCP 是通用底座，skill 是通用话术，原生插件只在个别 harness 有增量价值。**

一个 MCP server 可以接入所有支持 MCP 的 harness（Claude Code、Cursor、Trae、Codex、Cline、Zed、JetBrains、Gemini CLI……）；一份 SKILL.md 在 Claude Code 和 Codex 之间几乎零成本互用——两者的 skill 格式完全同构（YAML frontmatter + Markdown 正文），只是存放路径不同（`~/.claude/skills/` vs `~/.agents/skills/`），复制一份就能两边生效。

所以我给 oed 做接入时，重心分配是：MCP server 一份打全平台，SKILL.md 一份复制两处，原生插件只给 DeepSeek dsh 这种有独特运行时的 harness 单独写了一个（进程内工具省一层管道）。

## 二、工具层：MCP server 比你想的简单得多

本地 stdio 形态的 MCP server，本质就是一个 newline-delimited 的 JSON-RPC 2.0 循环。必须实现的方法只有四个：

```
initialize   → 回 protocolVersion + capabilities + serverInfo
tools/list   → 工具数组：{ name, description, inputSchema }
tools/call   → 按 name 分发 → 执行 → { content, isError }
ping         → 回 {}
```

我零依赖手写了一个（约 250 行 Node），没用官方 SDK——协议就这么大。三个实战要点，都是付过学费的：

**1. stdout 是协议通道。** 所有日志必须走 stderr。一条 print 写错地方，客户端解析直接失败，而且报错很难定位到你的日志行。

**2. 判断「是否被直接执行」要用 realpath 比对。** npm bin 和 npx 都是经符号链接启动的，`argv[1]` 不等于 `import.meta.url`。有些 harness 会把你的包当模块 import 进去（比如 dsh 的插件安装流程），没有「仅直接执行时启动」守卫的话，stdio 循环会把宿主进程卡死。

**3. 工具的 description 是模型唯一能「读到」的文档。** 把避坑规则直接写进去，比如「这个字段是网关托管的，永远不要传」「body 字段只进 body 不进 query」。模型不会去读你的 README，但每次调用前都会看到 description。

工具设计上有一个好用的模式——**三个工具封顶的探索-执行环**：

```
<x>_services  → 列出所有能力
<x>_describe  → 查某个能力/操作的契约（参数、必填项）
<x>_call      → 真正执行
```

模型有了探索环就能自学：先看有什么，再看怎么调，最后调用。你不需要把每个 API 的用法写死在 prompt 里。配套两个细节：永不解析数据源输出（CLI 的 JSON 原样透传给模型，它自己能读）；单调用超时和缓冲区放大一点（服务列表类响应可能很大）。

还有一个消灭前置依赖的技巧：包装类 MCP server 的死穴是「用户还得先装底层 CLI」。我的做法是 server 首次调用时自动解析 runner——PATH 上有就直接用；没有就 `uvx --from <pkg> <cli>` 临时拉一个（不改系统环境）；再没有就一次性 `pip install --user`。当然要留环境变量 opt-out——自动装包等于替用户做决定，必须给洁癖环境一个开关。

## 三、指令层：SKILL.md 怎么写

Claude Code 和 Codex 的 skill 格式同构，写法上三个要点：

1. **description 决定触发**。写清「什么时候该用 / 什么时候不该用」——这是唯一进入模型常驻上下文的字段，比正文重要；
2. **正文写工作流与避坑，不写 API 手册**。API 契约是 describe 工具的职责，skill 里重复一遍只会过期；
3. **保持框架中立**。正文只讲「装哪 + 命令 + 契约」，具体 harness 的接入方式放分发文档里。

## 四、auth：本地和远程是分水岭

四种模式一张表：

| 模式 | 机制 | 适用 |
|---|---|---|
| 匿名 | 无 | 公共只读数据（我的场景里 CVE、包、论坛全在此档） |
| 设备流 + keyring | 终端打印 URL → 用户浏览器授权 → CLI 轮询拿 token → 存 OS keyring | 本地 agent，用户级私有凭证 |
| PAT | 用户自己申请、填配置 | 过渡方案 |
| MCP OAuth 2.1 | server 做资源服务器，客户端弹浏览器授权，服务端存多用户 token | 远程 MCP（云端 agent、团队共享） |

一个反直觉的结论：**设备流登录不能搬进 agent 会话。**

在对话里让模型执行 `oed login` 会卡死：device flow 需要持续轮询几分钟等人工浏览器授权，而 harness 的工具调用是「执行 → 等退出 → 返回」，一两分钟就超时了，授权 URL 困在子进程缓冲里根本到不了用户眼前。

所以结论是：**登录是一次性的终端动作，使用是无交互的工具调用，两者永不混在同一会话。**本地凭证登录一次进 keyring，之后所有 harness 里的调用都只读 keyring。

而远程 MCP（豆包这类云端 agent 需要的形态）必须走 OAuth 2.1 + 服务端多用户凭证托管，那是一个独立的服务端项目——没有真实需求之前，别提前启动。

## 五、分发渠道全景（全部一手核实，2026-10）

这是本文最值钱的部分。先说总原则：

**npm/PyPI 包本身就是分发。**所有 harness 的配置最终都归结为「用包管理器拉起你的 server」——README 第一屏写清各客户端接入命令，这本身就是一个渠道。

**官方 MCP Registry（registry.modelcontextprotocol.io）是中立索引**，而且不是 PR 提交：用官方 `mcp-publisher` CLI 从项目里发布 `server.json`，GitHub OAuth 认证，域名验证走 DNS/HTTP challenge。半天能搞定，强烈建议做。

然后是各家 harness 的官方渠道，按门槛从低到高：

| harness | 渠道 | 门槛 |
|---|---|---|
| Gemini CLI / Qwen Code | gallery 自动收录 | **零门槛**：公开 GitHub 仓 + 打 topic + 根目录放 manifest，每日爬取，过校验即上架，无需任何提交动作 |
| DeepSeek dsh | npm 即渠道 | `dsh plugin add <npm包>`，包内声明注入配置，零审核 |
| Claude Code | plugin marketplace（git 仓库制） | 仓库里放 marketplace.json，用户自己 add，无审核 |
| Codex | ① 官方 portal（审核制）② git marketplace（自助） | portal 要组织身份验证 + ZIP 上传 + 接一个远程 MCP + 测试材料（账号、正反用例、演示视频）；**但 Claude 插件可以直接上传自动转换**；② 零审核 |
| Cline | 内置 marketplace | GitHub **issue** 提交（不是 PR）：仓库 URL、400×400 logo、两个自测勾选，人工安全审核；约 3000 个 open issue 积压，预期慢 |
| Cursor | 内置精选目录 | **无公开自助上架**，只能靠包仓库 + 用户手动 mcp.json |
| Trae | 内置市场 | **无任何公开提交通道**（已核实）。自助手段只有自生成安装深链（导入而非上架） |

注意 Codex 和 Gemini 那两行，是这轮调研里最有价值的发现：

- **Codex 官方 portal 直接接受 Claude 插件上传并自动转换**——manifest 双层结构：跨厂商的 `plugin.json`（agent-plugins.org 标准）为主，`.codex-plugin/plugin.json` 为兼容兜底。一份插件多 harness 分发，从愿景变成了现实。
- **Gemini CLI 的 gallery 是全自动收录**——不需要向任何人申请，把仓库按规范放好，被爬到就上架。这是最容易上的「官方渠道」。

C 端智能体平台（豆包背后那种形态）是另一条范式：

| 平台 | 状态（已核实） |
|---|---|
| 扣子 Coze | 插件商店上架流程有文档，OpenAPI 3.0 导入零代码，auth 平台托管；分发到微信/飞书/抖音/手机厂商智能体广场。**注意：现行渠道清单里没有豆包**——「发布到豆包」的自助通道不存在，豆包只作为企业版渠道互通的示例出现 |
| 阿里百炼 | 发布渠道 = Web 链接/钉钉机器人/微信公众号/API；无公开插件市场、无审核流程记录 |
| 腾讯元器 | 平台在线，但插件入驻细节在 JS + 登录墙后；现场画廊只有微信公众号智能体 |
| 百度文心智能体平台 | 爬虫封闭，无法评估 |
| Quora Poe | 创作者平台活跃、有按量分成；但官方文档 MCP 零提及，之前流传的「Poe 支持 MCP」不可证实 |

两大范式对照：开发者工具 harness 走 **MCP**，C 端智能体平台走**平台插件商店（OpenAPI 导入）**。完整覆盖 = MCP 打前者 + 一份 OpenAPI spec 打后者——扣子、GPT Actions、Copilot 的 API plugin 全吃 OpenAPI 3.0，一份 spec 可以同时在所有这些平台生成插件，零代码。

## 六、上架优先级（可以直接照抄的决策）

1. **npm/PyPI 发包 + README 写成 quickstart**——第一屏就是各客户端接入命令，这本身就是渠道；
2. **官方 MCP Registry** 发布（`mcp-publisher`，GitHub OAuth 认证即可）；
3. **零审核的原生渠道立即上**：Gemini/Qwen gallery、Claude Code marketplace 仓库制、dsh npm 制；
4. **非官方目录批量提交**：mcp.so、Smithery、PulseMCP 各有入口，门槛低，全部做一遍的成本是一下午；
5. **需要审核或无通道的（Cursor、Trae、Codex portal、豆包）不主动投入**，等用户量证明需求或平台开放自助通道。

## 七、付过学费的坑（节选）

- dsh 的 patch 语义：顶层条目 = 覆盖已有条目，**新增条目必须 `- insert:`**，否则报 entry not found；
- `dsh plugin add` 会 import 包主模块——这就是第 2 节 realpath 守卫的由来；
- npm **网页表单**发布会生成 0.0.0 占位版本，和 CLI 发布冲突，得 unpublish；发布走 CLI；
- npm CDN 有 2-4 分钟延迟：发布后干净机器装不到新版，加版本号或 `?nocache=`;
- 非 TTY 终端无法 npm publish（通行密钥流程要浏览器），发布必须在用户自己的交互式终端；
- `uvx` 装出来的 CLI shim 依赖 `realpath`/`dirname` 在 PATH 上——极简沙箱测试环境要记得放行；
- GUI 启动的 IDE 不继承 `~/.local/bin`：pip --user 装的 CLI 在 MCP 子进程里 ENOENT，配置里用绝对路径。

## 八、调研方法：这些结论是怎么核实出来的

最后说方法论，因为比结论更稀缺。

**JS SPA 文档先试 `<docs域名>/llms.txt`。**扣子的文档是全前端渲染的，直接抓是空壳；走 `docs.coze.cn/llms.txt` 拿到全站索引后，所有页面都是服务端渲染，一次抓完。

**搜索引擎里有伪造的「官方页面」。**调研 Codex 时搜到过 `codex.openai.com/docs/plugins/api` 这种 URL，格式、内容看着都挺官方——不存在。引用任何结论前，必须抓到原文核对。

**平台文档迁移频繁，旧博客链接不可信。**扣子、Trae、钉钉的文档这轮全部换过域名，2025 年的教程链接（包括「某平台数据源是 mcp.so」这种说法）现行文档里已经无法证实。

所以全文的纪律是：每条结论要么一手核实（抓到官方原文或实测），要么明确标注「不可核实」。没有「我记得是」级别的结论。

## 结尾

回到开头那个 Trae 的场景：最后排查下来是问法触发不了工具（加上面板开关没开），换一个模型「编不出来」的问法——选训练数据里根本没有的 CVE——模型老老实实走了 oed_describe → oed_call，答案和 Claude Code、Cursor 里跑出来的完全一致。

完整的调研文档（含全部表格、每家平台的细节和踩坑记录）在这里：

- GitHub：https://github.com/Hourunze1997/agent-harness-plugins
- GitCode：https://gitcode.com/hourunze/agent-harness-plugins

可复用的资产（都是这次实战沉淀的开源模板）：

- **oed-mcp**（npm 0.2.0）：零依赖 MCP server 模板，含自动引导和 dsh 插件双形态；
- **dsh-plugin-oed**（npm 0.1.0）：dsh 原生插件模板。

如果这篇帮你省下了踩坑时间，欢迎去仓库点个 star；你在哪个平台上架时遇到了文档里没覆盖的情况，欢迎留言，我去核实后更新文档。

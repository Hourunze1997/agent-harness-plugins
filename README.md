# Agent Harness 插件体系建设完整方案

> 调研日期：2026-10-28。写作者视角：以 oed（openEuler Developer CLI）的实战接入为
> 案例底座，覆盖「怎么开发一款 agent 插件」与「往哪上架」。所有**一手验证**过的结论
> 直接陈述；仅有二手来源或未能核实的，明确标 **【待验证】**。
>
> 配套实战仓：`gitcode.com/hourunze/oed-mcp`（npm: oed-mcp）、
> `gitcode.com/hourunze/dsh-plugin-oed`（npm: dsh-plugin-oed）。

---

## 0. 一页结论

1. **插件体系的本质是三层**：指令层（教模型「什么时候用、怎么用」）+ 工具层（让模型「真的能调」）+ 分发层（让用户「装得上」）。三层的标准件分别是 SKILL.md、MCP server、包仓库 + 市场目录。
2. **MCP 是通用底座，skill 是通用话术**。做一个 MCP server，可以接入所有支持 MCP 的 harness（Claude Code、Cursor、Trae、Codex、Cline、dsh、Gemini/Qwen、Zed、JetBrains……）；一份 SKILL.md 在 Claude Code 和 Codex 之间**几乎零成本互用**（同一格式）。
3. **原生插件只在个别 harness 有增量价值**（如 dsh 的进程内插件），优先级低于 MCP。
4. **分发渠道全景**：npm/PyPI 是运输层（包即分发）；官方 MCP Registry 是中立索引（CLI 发布 + 域名验证）；各平台市场（Claude Code marketplace、Codex portal/自建 marketplace、Trae 市场、Cline marketplace、扣子插件商店）是曝光层，多数要审核，早期性价比低于把 npm 包 + README 写好。
5. **auth 是本地/远程分水岭**：本地 agent 用「设备流 + 系统 keyring」（零服务端）；远程/云端 agent（豆包类）必须走 MCP OAuth 2.1 + 服务端多用户凭证——这是一个独立的服务端项目，应从匿名档起步。
6. **数据源 CLI 化是一切的前提**：stdout 恒 JSON、退出码固定、`--help` 机器可读。没有这个底座，上面三层全是空中楼阁。

---

## 1. 背景与问题定义

### 1.1 一个真实案例：oed

openEuler 社区的在线服务（CVE 通告、软件包、论坛、邮件列表、SIG 文档、代码托管……）各自有 HTTP API。oed 把它们收敛成一条命令：运行时从网关发现服务拉 OpenAPI 规范，动态构建命令表。三个让它「agent 友好」的设计决定：

| 设计 | 价值 |
|---|---|
| stdout 永远是单个 JSON（日志走 stderr） | `oed ... \| jq` 恒安全；模型直接吃 JSON 不用解析散文 |
| 退出码固定 5 个（0/1/2/3/4） | 脚本与 agent 按码分流，不猜报错文案 |
| `--help` 输出也是 JSON（参数从 OpenAPI 推导） | agent 自助探索学会用法，无需预先内置知识 |

### 1.2 要解决的问题

- 用户怎么在**自己的 agent harness** 里用上这项能力？
- 开发者该把插件**做成什么形态**（skill？MCP？原生插件？全都要？）
- 做完之后**上架到哪里**——哪些是官方渠道，哪些是非官方渠道，各自门槛是什么？
- **auth 怎么办**——本地与云端为什么是两条完全不同的路线？

---

## 2. 总体架构：三层模型

```
┌─────────────────────────────────────────────────────────┐
│  分发层  npm/PyPI 包 → 客户端配置 → 市场目录（官方/非官方）      │
├─────────────────────────────────────────────────────────┤
│  指令层  SKILL.md / AGENTS.md / 规则文件                       │
│         （何时用、参数约定、避坑规则、工作流）                    │
├─────────────────────────────────────────────────────────┤
│  工具层  MCP server（通用） / 原生插件（个别 harness）           │
│         （tools/list + tools/call，stdio 或 HTTP）             │
├─────────────────────────────────────────────────────────┤
│  数据源  CLI（推荐） / 直连 API                                │
│         stdout-JSON + 固定退出码 + 机器可读 help                │
└─────────────────────────────────────────────────────────┘
```

**各 harness 对三层的支持矩阵**（✅=官方机制，➖=无此层，【待验证】=未一手核实）：

| Harness | 指令层 | 工具层 | 原生插件 | 官方分发渠道 |
|---|---|---|---|---|
| Claude Code | ✅ skills | ✅ MCP | ✅ plugins/marketplace | 插件市场（git 仓库制）|
| Codex CLI | ✅ skills（`.agents/skills`）| ✅ MCP | ✅ plugin（Agent Plugins 标准 + 官方目录）| 官方插件目录（portal 审核）+ git marketplace 自助 |
| Cursor | ✅ rules | ✅ MCP | ➖ | 内置精选目录（无公开自助上架）|
| Trae | ➖ | ✅ MCP | ➖ | 内置市场无公开提交通道（已核实），走手动配置/安装深链 |
| DeepSeek dsh | ➖（prompt 即配置）| ✅ MCP client | ✅ cordis bundle 插件 | **npm 即渠道**（`dsh plugin add`）|
| Cline | ✅ | ✅ MCP | ➖ | 内置 Marketplace（GitHub issue 提交，已核实）|
| Gemini CLI | ✅ | ✅ MCP | ✅ extensions（`gemini-extension.json`） | gallery **全自动收录**（见 §6.1）|
| Qwen Code | ✅（fork 自 gemini-cli）| ✅ MCP | ✅ extensions（`qwen-extension.json`，兼容 Gemini/Claude/Agent Plugins）| gallery 自动收录 + npm 包直装 |
| Zed | ➖ | ✅ MCP（context_servers）| ➖ | ➖ |
| JetBrains AI | ✅ | ✅ MCP | ✅ IDE 插件市场 | JetBrains Marketplace |
| VS Code | ✅ | ✅ MCP（gallery 对接官方 Registry）| ✅ 扩展市场 | VS Code Marketplace |
| 豆包（云端） | ➖ | ✅ 远程 MCP（HTTP+OAuth） | ✅ bot 插件（第三方无自助上架） | 扣子生态可达（飞书/微信/手机厂商）；**豆包直发无自助通道（已核实）** |

**结论：先做 MCP server（一层投入、全平台接入），指令层按需补 SKILL.md，原生插件只对 dsh 这类有独特运行时的 harness 做。**

---

## 3. 工具层：MCP server 开发

### 3.1 传输与形态选择

| 形态 | 传输 | 适用 | auth 模式 |
|---|---|---|---|
| 本地 stdio server | stdin/stdout，每行一条 JSON-RPC 2.0 | 本地 CLI agent（本方案默认） | 复用本机凭证（keyring/PAT） |
| 远程 streamable HTTP server | HTTP + SSE | 云端 agent、团队共享、豆包类 | MCP OAuth 2.1 / 匿名 / BYO token |

**本地优先的理由**：零部署、零秘钥管理、天然能用用户自己的凭证；代价是每个用户自己装。oed 的决策就是「本地专用，远程等需求」。

### 3.2 实现：两条路

1. **官方 SDK**（TypeScript/Python）：省心，带依赖树。适合协议细节多、要 resource/promption 等高级能力的场景。
2. **零依赖手写**（oed-mcp 路线，~250 行 Node）：MCP 的 stdio 传输就是一个 newline-delimited JSON-RPC 2.0 循环。必须实现的只有：

```
initialize        → 回 protocolVersion（回显客户端请求值最稳）+ capabilities + serverInfo
tools/list        → 工具数组：{ name, description, inputSchema }
tools/call        → 按 name 分发 → 执行 → { content: [{type:'text',text}], isError }
ping              → 回 {}
notifications/*   → 不回复（无 id）
错误约定          → -32700 解析错误 / -32601 方法不存在 / -32602 参数错误
```

手写路线的三个实战要点（全部踩过）：

- **stdout 是协议通道**，一切日志走 stderr；
- **npm bin/npx 经符号链接启动**，判断「是否被直接执行」必须用 realpath 比对（`import.meta.url` vs `realpathSync(argv[1])`），否则被 harness 作为模块 import 时会误启 stdio 循环把宿主卡死；
- **工具 description 承载避坑规则**（「forum 的 Api-Key 是网关托管、永远不要传」「body 字段只进 body 不进 params」）——这是模型唯一能「读到」的文档。

### 3.3 工具设计原则（从 oed 实战提炼）

- **3 个工具封顶的探索-执行环**：`<x>_services`（列能力）→ `<x>_describe`（查契约）→ `<x>_call`（执行）。模型有了探索环就能自学，不用你在 description 里写死每个 API。
- **永不解析数据源输出**：CLI 的 JSON 原样透传给模型；退出码非 0 时包一层 `isError: true`，把错误 JSON 也交给模型（它能读懂并自我修正）。
- **超时与缓冲**：单调用超时 180s、maxBuffer 32MB 起（服务列表类响应可能很大）。
- **调用成本透明化**：`dry_run` 参数让模型先预览请求再真发，省真调用。

### 3.4 自动引导（bootstrap）：消灭「先装 CLI」的前置

包装类 MCP server 的死穴是「用户还得先装底层 CLI」。oed-mcp 0.2.0 的解法——server 首次调用时按序解析 runner：

```
PATH 上有 oed?  ──是──→ 直接用
      │否
有 uvx?        ──是──→ uvx --from <pkg> <cli>   （临时 venv，不改系统，首次自动从 PyPI 拉）
      │否
有 pip?        ──是──→ 一次性 pip install --user <pkg> → 重查 PATH / ~/.local/bin
      │否
报错：给模型可转述的修复指令
```

- 开关：`OED_MCP_AUTO_INSTALL=0` 关闭自动化（尊重洁癖环境），缺 CLI 变成清晰报错；
- 成功缓存 / 失败不缓存——用户中途装好 CLI，下一次调用即生效；
- 代价：自动装包 = 替用户做决定，必须留 opt-out；
- 沙箱测试坑：uvx 装出的 CLI shim 依赖 `realpath`/`dirname` 在 PATH 上。

---

## 4. 指令层：SKILL.md

### 4.1 格式即标准

Claude Code 与 Codex 的 skill 格式**完全同构**（YAML frontmatter 的 `name` + `description` + Markdown 正文），存放位置不同：

| | Claude Code | Codex |
|---|---|---|
| 用户级 | `~/.claude/skills/<name>/` | `~/.agents/skills/<name>/` |
| 仓库级 | `.claude/skills/<name>/` | `.agents/skills/<name>/` |
| 显式调用 | `/skill-name` | `$skill-name` 或 `/skills` |
| 自动触发 | description 匹配 | description 匹配（可关） |

**一份正文，两处软链/复制即可**。Trae/Cline 等没有 skill 层的 harness，靠 MCP 工具 description 承担同等职责。

### 4.2 写法要点

- `description` 决定触发：写清「什么时候该用 / 什么时候不该用」，这是唯一进模型常驻上下文的字段；
- 正文写**工作流与避坑**，不写 API 手册（那是 `describe` 工具的职责）；
- 保持「框架中立」：正文只讲「装哪 + 命令 + 契约」，具体 harness 的接入方式放分发文档。

---

## 5. auth：本地与远程的分水岭

### 5.1 四种模式

| 模式 | 机制 | 适用 | 成本 |
|---|---|---|---|
| **匿名** | 无 | 公共只读数据（oed 的 CVE/包/论坛/搜索全在此档） | 零 |
| **keyring + 设备流**（本地首选） | `oed ag login`（RFC 8628）：终端打印 URL → 用户浏览器授权 → CLI 轮询拿 token → 写 OS keyring；之后所有调用（含各 harness 里的）只读 keyring | 本地 agent，用户级私有凭证 | 零服务端（复用平台现成 token 端点） |
| **PAT / BYO token** | 用户自己申请 token，填进配置 | 本地或远程，过渡方案 | 低 |
| **MCP OAuth 2.1**（远程标配） | MCP server 是 OAuth 资源服务器；客户端首次连接弹浏览器授权；**服务端**存多用户 token | 远程 MCP（豆包类云端 agent、团队共享） | **高**：托管 + 用户体系 + 秘密存储 + 平台侧 OAuth 客户端注册 |

### 5.2 为什么设备流不能搬进 agent 会话

在 dsh/Claude Code 对话里让模型执行 `oed ag login` 会卡死：device flow 需要**持续轮询数分钟等人工浏览器授权**，而 harness 的工具调用是「执行→等退出→返回」（有 1-3 分钟超时），URL 困在子进程缓冲里到不了用户眼前。**结论：登录是一次性的终端动作，使用是无交互的工具调用，两者永不混在同一会话里。**

### 5.3 网关托管凭证（特殊但有价值的模式）

oed 论坛服务的 `Api-Key`/`Api-Username` 由**网关在转发时自动注入**，客户端（和模型）根本看不见这个参数。如果你的平台有网关层，把公共凭证收敛到网关是最干净的匿名化手段——工具 schema 里就不该出现这类参数。

### 5.4 远程化的决策（oed 现状）

远程 MCP + 分级 auth（匿名档先行 → ag 操作再 OAuth）是清晰路径，但这是一个**服务端项目**（托管、多用户凭证加密存储、与 AtomGit/openEuler 平台侧谈 OAuth 客户端注册——AtomGit API 目前是 PAT 模式）。触发条件：出现真实的云端 agent 需求（如豆包用户）。

**注意一条旁路（已核实并修正）**：字节系官方开发者平台是扣子（Coze），插件支持**直接导入 OpenAPI 3.0 spec、auth 平台托管**——匿名档服务不写代码即可上架扣子插件市场，分发到飞书/微信/抖音/手机厂商智能体渠道。但**现行文档中「发布到豆包」没有自助通道**（豆包仅作为企业版渠道 OAuth 互通示例出现）——豆包覆盖要么走企业渠道合作，要么等远程 MCP；远程 MCP server 项目维持按需启动。

---

## 6. 分发层：官方渠道 vs 非官方渠道

### 6.1 渠道全景（按门槛从低到高）

**第 0 层：包仓库 = 运输层（必做）**

- npm（Node 系）/ PyPI（Python 系）。**包本身就是分发**——所有 harness 的配置最终都归结为「用包管理器拉起你的 server」。
- 实战要点：`files` 白名单锁内容；发布走 CLI（网页表单会生成 0.0.0 占位版本还得删）；非 TTY 终端发不了（通行密钥要浏览器），CDN 有 2-4 分钟延迟。

**第 1 层：官方 MCP Registry（中立索引，强烈建议）**

- `registry.modelcontextprotocol.io`，「MCP 的 app store」，2025-09 上线预览。
- **不是 PR 提交**：用官方 `mcp-publisher` CLI 从你的项目里发布 `server.json`；认证支持 GitHub OAuth（交互）与 GitHub OIDC（CI）；namespace 归属验证用 DNS 或 HTTP challenge（如 `io.github.<user>/xxx` 要证明 GitHub 用户身份，`<域>/xxx` 要证明域名控制权）。
- 任何人可发，公开预览阶段。另有 **GitHub MCP Registry**（github.com/mcp，curated 列表，public preview，约 650 server）作为第二中立索引。

**第 2 层：各 harness 官方渠道**

| 渠道 | 机制 | 门槛 | 状态 |
|---|---|---|---|
| Claude Code plugin marketplace | git 仓库制：仓库里放 `.claude-plugin/marketplace.json`，用户 `/plugin marketplace add <repo>` 后安装 | 自助，无审核（官方 marketplace 另说） | ✅ 可立即用 |
| Codex plugin | **两条路，均已核实**：① 官方公共目录（ChatGPT/Codex 共用）：platform.openai.com/plugins portal 提交——组织身份验证 + ZIP 包上传 + 接一个远程 MCP（公网 Streamable HTTP，`/.well-known/openai-apps-challenge` 域名验证）+ 审核材料（测试账号、5 正 3 反用例、演示视频）；**不支持 API-key/Bearer auth 的 MCP**；skills-only 插件之后不能补 MCP ② git marketplace 自助：`$REPO_ROOT/.agents/plugins/marketplace.json` + `codex plugin marketplace add owner/repo`，零审核。manifest 双层：跨厂商 `plugin.json`（agent-plugins.org 标准）为主，`.codex-plugin/plugin.json` 为 OpenAI 兼容兜底。**Claude 插件可直接上传转换**（portal 接受 `.claude-plugin/plugin.json` + skills，自动转成 Codex 形态；marketplace 列表资质不迁移） | ① 审核（材料清单见前）② 零审核 | ✅ 两条都可立即用 |
| Cursor | 内置精选目录 | **无公开自助上架**，靠团队 curated | ➖ 只能第 0/1 层 + 用户手动 mcp.json |
| Trae | 内置 MCP 市场：官方仅描述为「社区热门 server」的 curated 列表，**无任何公开提交通道（已核实）**——文档（已迁 docs.trae.cn）只写「从市场添加/手动添加」，官方明言不审查第三方 server；2025 年「数据源为 mcp.so」的说法在现行文档中**不可证实**。开发者可用的自助机制只有**自生成安装深链**（`trae-cn://trae.ai-ide/mcp-import?type=…&name=…&config=…`，导入而非上架）；火山引擎 MCP Server 市场存在但无发布入口 | 无通道 | ➖ 手动 mcp.json / 安装深链 / 第三方目录曝光 |
| Cline | 内置 MCP Marketplace（cline.bot/mcp-marketplace） | **GitHub issue 提交（已核实，不是 PR）**：`github.com/cline/mcp-marketplace` 的 issue 模板——必填服务器仓库 URL、**400×400 PNG logo**、收录理由、两个自测勾选（「仅凭 README.md/llms-install.md 即可被 Cline 完成安装」「稳定可公开使用」）；人工安全审核（金融/加密货币类加强核查）；**不需要 smithery.yaml**。注意：仓里约 3000 个 open issue，周转可能慢 | 人工审核 | ✅ 可提交，预期慢 |
| Gemini CLI / Qwen Code | **gallery 全自动收录（已核实）**：geminicli.com/extensions 每日爬取——上架只需 ① 公开 GitHub 仓 ② 打 `gemini-cli-extension` topic ③ 根目录放 `gemini-extension.json`（manifest 含 mcpServers/contextFileName/excludeTools/settings；目录约定 commands/、hooks/、skills/、agents/），过校验即上架，无需任何提交动作。官方范例组织 `github.com/gemini-cli-extensions`（48 个仓）。**Qwen Code 已核实为超集**：原生 `qwen-extension.json`，自动转换 Gemini 格式；安装源额外支持 scoped npm 包、**Claude Code marketplace 直装**（自动转 claude-plugin.json）、Qoder 插件、**Agent Plugins v1 原生加载**；`/extensions manage` 全 UI 热重载 | 零门槛（自动爬取+校验） | ✅ 最容易上的官方渠道 |
| JetBrains / VS Code | 各自插件市场；VS Code 的 MCP gallery 对接官方 Registry | 常规市场审核 | ✅（经第 1 层间接覆盖） |
| 豆包 / 扣子 | **扣子插件商店（已核实，docs.coze.cn 服务端渲染可抓）**：插件上架 = 社区→插件市场→上架插件（或资源库→上架插件），提交描述/使用说明/类别；格式含 OpenAPI 3.0 导入（零代码）/代码插件；auth 平台托管（无需鉴权/service/OAuth）。另有技能商店（公开审核自查指南与拒审案例）与作品社区；变现 = 付费模板/付费技能/受邀制付费插件。**重要修正：现行发布渠道清单里没有豆包**——渠道为 API/Chat SDK/企业商店/微信系/飞书系/抖音小程序/掘金 + OPPO/vivo/小米/荣耀/联想/中兴智能体广场，豆包仅作为企业版「渠道 OAuth 账号互通」的示例出现 | 平台审核（时长未公开） | ✅ 扣子生态（飞书/微信/手机厂商）可立即尝试；豆包直发无自助通道 |
| DeepSeek dsh | **npm 即渠道**：`dsh plugin add <npm包>`，包内 `dsh.bundle.patch` 声明要注入的配置 | 自助，零审核 | ✅ 可立即用 |

**第 3 层：非官方目录（曝光补完）**

- [mcp.so](https://mcp.so)、[Smithery](https://smithery.ai)（要 `smithery.yaml`）、[PulseMCP](https://pulsemcp.com)、[Glama](https://glama.ai) —— 各有提交入口，收录门槛低；
- GitHub awesome 列表（如 awesome-mcp-servers，26k+ star）—— PR 收录；
- 价值排序：官方 Registry > mcp.so/Smithery > awesome 列表。都做一遍的成本是一下午，值得。

### 6.2 C 端智能体平台（各家「扣子等价物」）

豆包背后的扣子不是孤例——每家大厂都有同构的平台：**开发者平台搭 agent/插件 → 平台审核 → 分发到自家 C 端产品**。

| 母公司 | 开发平台 | 插件格式 | 分发到的 C 端 | 可信度 |
|---|---|---|---|---|
| 字节 | 扣子 Coze（coze.cn，已核实 docs.coze.cn） | OpenAPI 导入 / 代码 / MCP | 飞书、微信、抖音、掘金、手机厂商智能体广场；**豆包无自助通道（已核实）** | ✅ 通道已核实（豆包除外） |
| OpenAI | GPT Store + Apps SDK（Apps in ChatGPT，MCP-based） | Actions（OpenAPI）/ Apps SDK | ChatGPT、Codex | 机制确认；提交通道与 Codex 官方目录共用 platform.openai.com/plugins portal（已核实，§6.1） |
| Anthropic | Claude Connectors 目录 + Claude Code marketplace | 远程 MCP / git 仓库制 plugins | Claude.ai、Claude Code | ✅ 一手核实 |
| 微软 | **双路（已核实）**：① Copilot Studio agent → Teams 应用店分区（Built with Power Platform / 经管理员批准的 Built for your org）/ M365 Agent Store / 经 Seller Dashboard 认证上架 Teams Store 与 Microsoft Marketplace；② M365 Copilot 插件（声明式 agent + skills + connectors + MCP 集成；声明式 agent 内含 **API plugin（OpenAPI 导入）**与 MCP plugin 动态发现）→ 经 Partner Center「Microsoft 365 and Copilot」认证上架 Microsoft Marketplace，或组织内/admin 上传/直分享（`wiqd plugin share`）。Copilot Studio 原生支持挂 MCP server（跨租户需 connector 认证） | 认证审核（Teams Store 自动+人工，约 24 工作时反馈）| ✅ 企业向，流程重 |
| GitHub | **Copilot Extensions（GitHub App 制）已于 2025-11-10 退役**，被 MCP 取代。现行（已核实）：① MCP server 全表面支持 + **GitHub MCP Registry**（github.com/mcp，curated，public preview，约 650 server）② **Agent Plugins 1.0 GA**（2026-08，即 agent-plugins.org 开放标准）——经 plugin marketplace（Git 仓 + marketplace.json）分发，官方默认启用 `github/copilot-plugins` 与 `github/awesome-copilot`，上架=向 marketplace 仓提 PR，**无 GitHub 侧审核**（企业可用 allowlist 管控）③ Agent apps（伙伴通道，waitlist） | ① preview ② PR 自助、无审核 | ✅ 自助路径畅通 |
| Google | Gemini Enterprise/Agentspace；Gemini CLI extensions | MCP connector / extension 包 | Gemini 企业版 / CLI | CLI 侧已核实（§6.1 gallery 全自动收录）；企业版通道【待验证】 |
| 腾讯 | 元器（**已核实 2026-10 仍在线**，SSR 可抓） | 插件形态无法核实（子页全 JS+登录墙；「停服」传闻与在线现状矛盾、不可证实） | 微信公众号智能体（现场画廊仅此渠道） | 平台活、细节不可得 |
| 阿里 | 百炼（help.aliyun.com **已核实**）+ 钉钉 AI 助理（JS 墙不可核实） | 智能体应用 + 自定义插件（API 集合）+ MCP；发布 = Web 链接/钉钉机器人/微信公众号/API（Agent 2.0 仅 API） | **无公开插件市场、无审核流程记录**；百炼→钉钉机器人发布有文档 | 平台活、无市场 |
| 百度 | 文心智能体平台（agents.baidu.com 在线但爬虫封闭，仅 meta 自述「流量分发路径+商业闭环」）；企业线并入千帆 | 不可核实 | 百度系流量 | 封闭，不可评估 |
| Quora | Poe 创作者平台（creator.poe.com，**已核实**） | Prompt/Server/Script Bots、Canvas Apps、API Bots；变现 = Bot Monetization API（按消息计费） | Poe 目录 | 官方文档 **MCP 零提及**（「Poe 支持 MCP」的旧说法不可证实） |
| 开源自托管 | Dify（v1.0 起自带插件市场）、FastGPT、Coze Studio | 工具插件 DSL、MCP | 自有部署实例 | ✅ |

两个横贯性观察：

1. **OpenAPI 导入是 C 端平台的通用方言**——扣子、GPT Actions、Copilot API plugin、元器、Dify 全吃 OpenAPI 3.0。spec 驱动的数据源（如 oed）一份 spec 可以同时在所有这些平台生成插件，零代码。
2. **两大分发范式**：开发者工具 harness（Claude Code/Cursor/Codex/dsh…）走 **MCP**；C 端智能体平台（豆包/ChatGPT/M365）走**平台插件/agent 商店**。完整覆盖 = MCP 打前者 + spec 导入打后者。

3. **插件 manifest 正在跨厂商标准化，且已落地**：agent-plugins.org 的 Agent Plugins 规范（根目录 `plugin.json` + 固定位置的 `skills/`、`mcp.json`、`hooks/`），TSC 含 Amazon/Cursor/Microsoft/OpenAI/Vercel 维护者。**GitHub 已于 2026-08 将 Agent Plugins 1.0 GA 到 VS Code Copilot / Copilot CLI / Copilot app**；OpenAI 在其上做 `.codex-plugin/plugin.json` 兼容层且 portal 直接接受 Claude 插件转换上传；Qwen Code 原生加载 Agent Plugins v1、可直接从 Claude Code marketplace 安装。互转矩阵已经出现：Claude ↔ Codex（portal 转换）、Gemini → Qwen（自动转换）、Agent Plugins → GitHub/Qwen（原生）。「一份插件、多 harness 分发」从愿景变成了现实——manifest 设计应对齐该标准。


### 6.3 上架策略建议（优先级即顺序）


1. npm/PyPI 发包 + README 写成 quickstart（第一屏就是各客户端接入命令）——**这本身就是「渠道」**；
2. 官方 MCP Registry 发布（`mcp-publisher`，GitHub OAuth 认证即可）；
3. 有原生插件体系且零审核的（Claude Code marketplace 仓库制、dsh npm 制）立即上；
4. 非官方目录批量提交；
5. 需要平台审核/无公开通道的（Cursor、Trae、Codex curated、豆包）**不主动投入**，等用户量证明需求或平台开放自助通道。

---

## 7. 从零到上架：完整流程清单

**第 1 步：数据源 CLI 化**（如果还没有）
- stdout 恒 JSON / 退出码固定 / `--help` 机器可读 / 发现机制让能力可枚举；
- 参照 oed：`dynamic.py` 从 OpenAPI 推导命令与 per-param flag，缓存 10 分钟。

**第 2 步：MCP server**
- 三个工具探索-执行环（services/describe/call 模式）；
- 手写 JSON-RPC 或官方 SDK；description 里写死避坑规则；
- 自动引导（如需要）；协议测试清单：握手 / tools/list / dry-run / 真实调用 / 未知工具 -32602 / 坏行 -32700 / 退出码→isError 透传。

**第 3 步：skill 层（可选但便宜）**
- 一份 SKILL.md，frontmatter 触发描述 + 工作流正文；复制到 `~/.claude/skills/` 与 `~/.agents/skills/`。

**第 4 步：本地端到端验证**
- 每类 harness 用同一组测试问法（如「查 CVE-2019-10082 的公告编号与严重程度」），核对工具调用链与答案一致性——oed 的基准答案：openEuler-SA-2020-1002 / Medium；
- 问法要选模型**编不出来**的（避免模型用训练数据蒙对，掩盖工具没被调用）。

**第 5 步：分发**
- 发包 → README quickstart → MCP Registry → 平台市场（按 §6.2 优先级）→ 非官方目录。

**第 6 步：进阶（按需）**
- 原生插件：仅当目标 harness 有独特运行时价值（如 dsh 进程内工具省一层管道）；
- 远程化 + OAuth：见 §5.4，触发式投入。

---

## 8. 安全清单（发布前自查）

- [ ] 工具入参校验后再用（inputSchema 声明 + server 侧再验）；
- [ ] 不把用户内容未清理地拼进第三方 API 调用（防提示注入外泄）；
- [ ] 每个外部调用有超时；错误以结构化形态返回（退出码/isError/retryable 提示），不留半截输出；
- [ ] 凭证永不进工具输出、永不进日志；本地凭证只走 OS keyring；
- [ ] stdout 纯净（协议通道），日志全走 stderr；
- [ ] 自动安装行为有环境变量 opt-out，并在文档明示。

---

## 9. oed 案例附录

### 9.1 已验证接入矩阵（同一问法，答案交叉一致）

| Harness | 形态 | 状态 |
|---|---|---|
| Claude Code | skill + MCP | ✅ |
| Cursor CLI 2026.10 | MCP（`~/.cursor/mcp.json`，CLI/IDE 同读） | ✅ |
| dsh | 原生插件 + MCP client 两路 | ✅ |
| Trae | MCP（手动 mcp.json） | ⚠️ 用户侧安装成功、模型未调工具（排查：面板状态/Agent 模式/问法/PATH） |
| Codex CLI | skills + MCP | 配置路径已核实，端到端待实测 |

### 9.2 踩坑清单（每一坑都真实付过学费）

1. **dsh patch 语义**：顶层条目=覆盖已有条目；**新增条目必须 `- insert:`**，否则报 entry not found；
2. **dsh plugin add 会 import 包主模块**：入口必须加「仅直接执行时启动」守卫（realpath 比对）；
3. **npm 网页表单发布**会生成 0.0.0-stage 占位版本，与 CLI 发布冲突，需 unpublish；
4. **npm CDN 延迟 2-4 分钟**：发布后干净机器装不到新版，加 `?nocache=` 或显式版本号；
5. **非 TTY 无法 publish**（通行密钥流程），必须用户交互终端；
6. **uvx 装的 CLI shim 依赖 coreutils**（realpath/dirname）在 PATH 上，极简沙箱测试要放行；
7. **GUI 启动的 IDE 不继承 `~/.local/bin`**：pip --user 装的 CLI 在 MCP 子进程里 ENOENT，配置里用绝对路径或改系统级安装；
8. **多凭据行的 credential 文件**：按行取 token（曾因 `tail -1` 把仓库建到错误的组织名下）。

### 9.3 复用资产

- `oed-mcp`（npm，0.2.0）：零依赖 MCP server 模板，含 auto-bootstrap 与 dsh bundle patch 双形态；
- `dsh-plugin-oed`（npm，0.1.0）：dsh 原生插件模板（name/inject/apply + ToolDefinition）；
- `skills/oed-cli/SKILL.md`：Claude Code/Codex 双兼容 skill 模板。

---

## 10. 待验证清单（诚实边界）

- ~~Cline marketplace 的当前提交流程~~ **已核实**（2026-10）：GitHub issue 提交（`cline/mcp-marketplace` 仓模板，非 PR），需 logo/自测勾选，人工安全审核，无需 smithery.yaml；约 3000 open issue 积压；
- ~~Trae MCP 市场的公开提交通道~~ **已核实**（2026-10）：**不存在**——curated、无入口、官方不审查第三方 server；mcp.so 数据源说法不可证实；自助手段仅安装深链；
- ~~Codex「curated directory」上架流程~~ **已核实**（2026-10）：portal 审核制 + git marketplace 自助制并存，细节见 §6.1；注意搜索引擎存在伪造的「官方」页面（如 codex.openai.com/docs/plugins/api、openai/codex-app），均不存在，以 developers.openai.com/plugins 系列为准；
- ~~Gemini CLI / Qwen Code extensions 的官方收录仓现状~~ **已核实**（2026-10）：gallery 全自动收录（公开仓 + `gemini-cli-extension` topic + 根目录 manifest，每日爬取过校验即上架）；Qwen 为超集（npm 直装/Claude marketplace 直装/Agent Plugins v1 原生）；注意官方组织是 `gemini-cli-extensions`（不是 `google-gemini/gemini-cli-extensions`，后者不存在）；
- ~~国内 C 端平台（元器/百炼/文心/Poe）~~ **已核实**（2026-10）：元器仍在线但细节不可得（JS+登录墙）、现场分发仅微信公众号；百炼发布渠道 = Web/钉钉机器人/微信公众号/API，**无公开插件市场**；文心智能体平台爬虫封闭不可评估；Poe 创作者平台活跃但**官方文档 MCP 零提及**；钉钉 AI 助理广场同样 JS 墙不可核实；
- ~~豆包/扣子生态~~ **已核实并修正**（2026-10，docs.coze.cn 服务端渲染可抓）：扣子插件商店上架流程有文档（审核时长未公开）；**现行渠道清单无豆包**——此前「bot 审核后可发布到豆包 app」的说法不成立，豆包仅作为企业版渠道 OAuth 互通示例出现。仍待验证：豆包侧第三方工具的现行政策；
- 各平台对**远程 MCP + OAuth** 的支持成熟度（豆包类是主要变量）。

以上任一项在投入前应先花 30 分钟核实官方文档，本文不为其背书。

**调研方法备注**（本轮验证沉淀）：① 多家平台文档是 JS SPA——先试 `<docs域名>/llms.txt`（扣子 docs.coze.cn 即此法突破，全站服务端渲染可抓）；② 搜索引擎结果里存在伪造的「官方页面」（Codex、Poe 均已确认假 URL），引用前必须抓原文核对；③ 平台文档迁移频繁（扣子→docs.coze.cn、Trae→docs.trae.cn、钉钉→open.dingtalk.com），旧博客链接不可信，以域名内当前页面为准。

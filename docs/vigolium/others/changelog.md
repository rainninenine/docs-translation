# 文档 API 参考 其他 更新日志 复制页面

# 发布说明与版本历史 - Vigolium

## 复制页面

Vigolium 的每个显著变更，按最新优先排列。每个条目对应 `@vigolium/vigolium` 包的一个已发布版本。

此页面镜像主仓库中的 `CHANGELOG.md`，该文件为权威来源。运行 `vigolium version` 查看已安装的构建版本，运行 `vigolium update` 进行升级。

---

## v0.2.1

**2026-07-05**

在 v0.2.0 稳定线之上的一次检测广度与 Agent 供应商发布。新增十一个模块（5 个主动 + 6 个被动），多个现有检测器增加了带外与绕过分支，Olium Agent 运行时新增三个供应商后端（公共 OpenAI Responses API、Claude Code Agent-SDK 桥接、通用 Anthropic Messages 代理）。同时引入一批读取命令的人体工程学改进——分组漏洞发现树视图、`--pick` 位置选择、可重复的 AND 组合 `--search`、默认紧凑（且语法高亮）的 Markdown、以及扫描后流量打印。注册表现为 198 个主动 + 115 个被动模块。

### 新增

**新主动模块（5 个）：**

- `xpath-injection`（高危）——通过引擎错误签名和布尔盲注检测 XPath/XQuery 注入。
- `sqli-out-of-band`（严重）——通过 OAST 回调（`LOAD_FILE` / `xp_dirtree` / `UTL_HTTP`）到每个载荷唯一子域名确认的带外 SQL 注入。
- `unauth-service-exposure`（严重）——通过 HTTP 暴露的未认证 Docker / Kubernetes / 数据存储 API。
- `tls-protocol-cipher-audit`（中危）——通过直接协商已弃用协议和弱密码套件对主机 TLS 进行评级。
- `session-fixation`（中危）——宽松的会话管理（服务器接受攻击者固定的会话标识符）。

**新被动模块（6 个）：**

- `llm-endpoint-fingerprint`（信息）——检测 LLM / AI API 端点，基于新的 `infra/llmsig` 签名目录。
- `payment-integration-audit`（信息）——标记支付网关集成风险面（Stripe、PayPal、Braintree、Square、Adyen、Razorpay、Checkout.com）。
- `css-injection-detect`（信息）——输入被反射到 CSS 上下文中。
- `dom-clobbering`（信息）——DOM 污染小工具：来自命名全局变量的 JavaScript 接收器。
- `cross-origin-isolation-audit`（信息）——缺少跨源隔离头（COOP / COEP / CORP）。
- `reverse-tabnabbing-detect`（低危）——`target=_blank` 链接缺少 `rel=noopener`。

- `vigolium skills` 命令——列出、检查（get）并安装内置于二进制文件中的编码 Agent 技能包到编码 Agent 的技能目录（`--agent claude|codex|agents`，`--scope project|global`，`--all`，`--dir`）。由于包是嵌入的，它始终与您安装的 CLI 版本匹配。参见“在您的 Agent 中使用 Vigolium”。
- `olium provider openai-responses`——公共 OpenAI Responses API（`POST /v1/responses`），使用 API 密钥认证。密钥解析方式与 `openai-api-key` 相同（`agent.olium.llm_api_key` → `$OPENAI_API_KEY`），仅在线格式上有所不同。默认模型 `gpt-5.5`。
- `olium provider anthropic-claude-sdk-bridge`——通过 Agent SDK 驱动 Claude Code，使用新的 `vigolium-audit` 桥接边车，利用您已登录的 Claude Code 订阅（无需密钥；显式的 `llm_api_key`/`oauth_token` 会被转发）。桥接二进制文件从 `agent.olium.bridge_binary` / `--bridge-bin` 解析，否则使用嵌入的 blob，否则使用 `PATH` 上的 `vigolium-audit`。`anthropic-claude-cli` 可作为 `anthropic-cli` 的别名接受。
- `olium provider anthropic-compatible`——任何兼容 Anthropic Messages（`/v1/messages`）的端点，位于自定义的 `agent.olium.custom_provider.base_url`（自托管 Claude 网关或 Messages 格式代理）；`extra_headers` 可切换认证方案。
- `scan --print-traffic` / `scan --print-traffic-tree`——扫描后，将运行的 HTTP 流量转储到标准输出：原始请求/响应对（如 `traffic --raw`）或主机/路径层次树（如 `traffic --tree`）。镜像 `--print-finding`，可用于 `scan` / `scan-url` / `scan-request` / `run`。
- `finding --tree`（也可用 `vigolium finding tree`）——主机/路径层次树，将重复的漏洞发现按标题+严重性折叠为一个 ×N 节点，列出每个受影响的 `#id`（置信度）→ URL。位置参数现在像 `traffic` 一样路由：`tree` 激活树视图，`ls`/`list` 选择表格，其他任何内容作为模糊搜索词。
- `finding --pick`——在过滤和排序后，将结果页面缩小到特定的基于 1 的位置（`--pick 2`，`--pick 1,3`，`--pick 2-4`）；与 `--raw` / `--burp` / `--markdown` / `--json` / `--tui` 组合使用。
- 可重复的 `--search` 应用于 `finding` 和 `traffic`——现在可重复且 AND 组合，因此每个添加的术语进一步缩小匹配范围。`finding search` 也扩展到覆盖模块名称和简短描述。
- `--bridge-bin` 标志 / `agent.olium.bridge_binary` 配置——将 `anthropic-claude-sdk-bridge` 供应商指向承载 SDK 桥接的特定 `vigolium-audit` 二进制文件。

### 变更

- JWT 漏洞扫描器新增两个伪造分支：`jku`/`x5u` 头声明注入指向唯一 OAST URL，以及 `kid` 路径遍历令牌强制 `alg=HS256` 并使用 `/dev/null` 的空内容签名。
- XXE 扫描器新增盲注/带外检测，通过每个载荷唯一的外部实体和参数实体 DTD 引用唯一 OAST URL。
- XML / SAML 安全新增签名绕过检测——剥离 `<Signature>` 元素并确认目标仍接受未签名文档。
- 敏感文件发现新增高置信度签名，用于完全可遍历的 `.git` 树、Terraform 状态（`*.tfstate`）和 docker-compose 文件，每个都带有 HTML 反标记。
- CSP 弱点审计现在理解 `nonce-`/`sha*-` 源表达式和 `strict-dynamic`，因此当 nonce/哈希或 `strict-dynamic` 使现代浏览器忽略 `unsafe-inline` 时，不再标记它。
- `finding --markdown` / `traffic --markdown` 默认压缩响应体——响应现在渲染为围绕漏洞发现的 `matched_at`/`evidence`（或前导预览）的窗口，而不是整个页面，不再依赖 `-S`/`--compact`；传递 `--full-body` 以完整渲染主体。请求始终完整显示。渲染的 Markdown 在交互式终端上语法高亮，在管道/重定向时保持纯文本。
- 子命令帮助重新组织——继承的全局标志块被重组，当标准输出被管道时折叠为单行指针；本地扫描标志帮助被重组为章节。
- 供应商覆盖标志列出新的后端——跨 agent 子命令的 `--provider` 帮助现在列出 `openai-responses`、`anthropic-compatible` 和 `anthropic-claude-sdk-bridge`；`vigolium agent --list-agents` 显示桥接 agent 和 `anthropic-claude-cli` 别名。

### 修复

- 路径规范化 / FUZZ 基线假阴性——内容发现指纹器不再对绕过路径运行 `path.Clean`；绕过序列（`%23`、`..`、`//`、`..;/`）在线路上逐字节保留，基线正确键控，因此依赖于非规范化路径的缓存键和内容发现漏洞发现再次被检测到。
- 范围关键字过度匹配——源范围关键字匹配现在与主机的可注册域前导标签（eTLD+1）比较，而不是裸主机名字符串，因此包含关键字的无关第三方子域（例如 `*.cloudflareaccess.com` SSO 墙）被正确排除。
- `--db-isolate` 在全新共享 `--db` 上的合并竞争——目标数据库临界区（打开 → 模式创建 → 默认种子 → 合并）现在完全在一个跨进程合并锁内运行，修复了在 `-P` 扇出到一个全新共享数据库时出现的 `SQLITE_BUSY` 不稳定问题。

---

## v0.2.0

**2026-07-05**

第一个稳定（非 beta）版本，检测覆盖范围大幅扩展。新增六个模块系列——GraphQL 安全测试、Adobe Experience Manager (AEM)、SaaS 数据暴露（Salesforce / ServiceNow / Power Pages）、Internet Information Services (IIS)、扩展的模型上下文协议 (MCP) 套件，以及独立的依赖混淆和 JavaScript 美化被动模块。它们共同增加了 29 个注册模块（22 个主动 + 7 个被动），使注册表达到 193 个主动 + 109 个被动。每个新的主动系列都经过技术门控、故障关闭和多轮确认，以保持低误报率。

### 新增

- **GraphQL 安全扫描（OWASP WSTG）**——`graphql_scan` 模块获得完整的阶段套件，基于新的 `pkg/graphqlx` 库：操作名称扩展、内省/控制台暴露、查询批处理、字段级授权（IDOR）、反射型 XSS 错误、布尔型 SQL 注入和拒绝服务（深度嵌套/别名/循环查询）。每个阶段仅针对确认使用 GraphQL 的端点运行（`{ __typename }` 探测返回规范形状的封装，而不仅仅是单词“graphql”），并在多轮中确认漏洞发现。
- **Adobe Experience Manager (AEM) 模块系列**——11 个主动 + 1 个被动 `aem_*` 模块，基于共享的、故障关闭的 `infra/aem` 门控（`ConfirmAEM`）：调度器绕过（差分）、敏感 Servlet 和控制台暴露、QueryBuilder 内容发现（将 `rep:password` 标记为严重）、默认凭据、CloudSettings 注入、反射型 XSS（无头确认）、SSRF、带外注入、XXE（CVE-2019-8086 实体扩展证明）和 RCE，以及一个被动 `aem-fingerprint`。仅检测；每个主动探测都门控到真实的 AEM 实例。
- **SaaS 数据暴露系列（Salesforce / ServiceNow / Power Pages）**——7 个主动 + 3 个被动模块，基于共享的 `infra/saasprobe` 门控：Salesforce Aura 对象/记录暴露（`getConfigData` / `getItems`）、Salesforce Aura 访客 Apex 执行（`ApexActionController.execute` 可达性，通过一个只读托管包方法确认——访客驱动 SSRF/DML/数据泄露的支点）、Salesforce Lightning 调试模式暴露（以 `PRODDEBUG`/`DEV`/`JSTESTDEBUG` 提供的站点，向未认证客户端泄露后端堆栈跟踪）、ServiceNow 小部件和 KB 文章数据暴露（`g_ck` + cookie，将 401 视为非漏洞发现）、以及 Power Pages Dataverse `/_api` 暴露，每个都配有一个按供应商的被动指纹。每个主动探测门控到实时 Aura 网关（故障关闭存在性检查），将全包阴性控制与阳性配对，并在多个独立轮次中确认。
- **IIS 模块系列**——`iis-shortname-discovery` 获得真正的 8.3 短名称解析（`gen8dot3`/校验和技术 的 Go 移植，为 deparos 内容发现提供嵌入词表），加入两个新的主动模块：`iis-cookieless-source-disclosure` 和 `iis-extension-confusion-bypass`（`::$DATA` / 尾随点）。所有模块均经过 IIS/ASPX 门控和多轮确认。
- **MCP 套件扩展**——两个新的主动模块（`mcp-tool-definition-drift`、`mcp-dos-amplification`）和一个新的被动模块 `mcp-dangerous-tool-exposure`，扩展了针对 OWASP 审计的模型上下文协议系列。
- **依赖混淆被动模块**——仅从第一方 JS 响应中提取作用域 npm 导入说明符，并标记任何在 `registry.npmjs.org` 上未认领（404）的包。可疑/暂定严重性，通过刷新器进行批量注册表查找，带有过度查询防护（公共作用域跳过列表 + 语句边界正则表达式），以及对任何不确定查找的故障关闭处理。
- **JS 美化被动模块**——将压缩或打包的第一方脚本（React/Next/Vue SPA 输出）解压缩并解包为可读源代码，由嵌入的 `jsscan` 工具通过 `webcrack`（静态解包器——无基于 eval 的反混淆）驱动。`jsscan` 作为有界 bun 子进程运行（上限为 `GOMAXPROCS-2`，最大 4，因此宽泛的被动扇出到许多 JS 记录不会为每个记录生成一个进程），并返回美化后的文档，对于 webpack/browserify 包，还返回带有恢复源路径的按模块拆分——此版本扩展了 `jsscan webpackExtractor` 以重建这些模块路径。对于 JavaScript 响应，存储的记录主体被原地重写为美化后的文档（丢弃 `Content-Encoding`），并标记为 `js-beautified` / `js-format:<format>`，因此 `linkfinder`、观察到的词收获和手动审查都读取干净的源代码；HTML 页面中的内联 `<script>` 代码仅被美化到漏洞发现证据中（`js-inline-beautified`），从不重写页面主体。一个廉价的 Go 端预门控（最小 500 字节，webpack/Next/SystemJS 标记或压缩的平均行长度）避免为微小或已可读的脚本生成子进程，结果按主机+路径去重，并且已知的第三方资产——CDN 库和分析/captcha/聊天/支付/机器人管理 SDK，通过 `jsvendor` 在主机、路径和内容签名上匹配——被跳过，除非扫描实际针对该供应商。信息严重性；当嵌入的二进制文件不可用时，模块干净地无操作。

### 变更

- 版本号去掉了 `-beta` 预发布标签，跳转到次要版本——`v0.2.0` 是第一个稳定版本，反映了此次覆盖扩展的规模。
- 现有 MCP 系列的误报加固——方法枚举阴性控制、批处理单例基线、会话固定门控、源局部性、服务器探测 `isError`/`skip` 处理、以及资源模糊 OAST/元数据检查——加上扩展的 `mcp-description-injection` 覆盖。

### 内部

- 新的共享库：`pkg/graphqlx`（GraphQL 检测、内省、模式和操作构建），以及支持 AEM 和 SaaS 系列的 `infra/aem` / `infra/saasprobe` 门控包。
- 在 `tech_tags.go` 中为新的门控添加了技术指纹标签（`aem/adobe`、`salesforce/lightning/aura`、`servicenow`、`powerpages/dataverse`），由各自的 `*_fingerprint` 被动模块发布。
- `jsscan webpackExtractor`（`platform/jsscan`）被扩展以恢复 `js-beautify` 的按模块源路径；支持性调整涉及爬虫（`pkg/spitolas`）、内容发现（`pkg/deparos`）和核心执行器/速率限制器。

---

## v0.1.47-beta

**2026-07-04**

一次发现深度和数据可移植性发布。爬虫现在使用从目标和页面本身派生的值填充表单（因此注册→登录流程携带一致的身份），并且在平衡/深度强度下，针对确认的本地登录表单喷洒一个简短的常见凭据列表以进行认证爬取。此外，`vigolium import` 学会将一个 vigolium `.sqlite` 结果数据库合并到另一个中，`agent audit -S` 可以将其报告与原始扫描输出捆绑，`vigolium server` 获得一个仅被动扫描的接收即扫描模式。

### 新增

- **爬虫响应感知表单填充**——爬虫（`pkg/spitolas`）现在从目标和响应派生表单值，而不是始终使用固定占位符：它回忆同一爬取中先前输入的值（因此注册身份在登录时重用），回退到目标派生值（`vigolium-crawl@<domain>`、`<label>_crawl`），然后回退到页面上的示例（字段的 `defaultValue` 或 `datalist` 选项），最后才使用现有的智能/固定默认值。密码从不从页面获取（保留一个强固定密码）。默认开启且非破坏性——没有填充上下文时，保留原始行为。
- **在确认的登录表单上进行常见凭据登录喷洒**——在平衡强度下，爬虫尝试一个最小列表（`admin:admin`、`admin:123456`），在深度强度下使用完整的文档化短列表（绝不是词表）针对登录表单，但仅在确认表单是真正的本地登录（恰好一个可见密码 + 提交 + 范围内操作，不是外部 IdP）并且一个虚假控制对被拒绝之后。成功时，会话 cookie 持续存在，爬取继续认证。在快速/轻量强度下关闭（锁定/授权风险）。通过爬虫结果上的 `LoginCredsTried`/`LoginCredsSucceeded` 报告。
- **`vigolium import` 合并 vigolium SQLite 数据库**——`import` 现在接受另一个 vigolium 结果数据库作为其源（通过其 SQLite 魔术头自动检测，因此 `.sqlite`/`.sqlite3`/`.db` 或裸名称均可），并将其折叠到目标 `--db`（或配置的默认值）中。一个无损、幂等的 SQLite→SQLite 合并，包括 HTTP 记录、漏洞发现、扫描、Agent 扫描和 OAST 交互，按自然键去重——重新导入同一数据库是无操作的，每行保留其原始 `project_uuid`。与 `scan -S --format sqlite` 的自然伴侣：扇出每个主机的 `.sqlite` 文件，然后将它们合并回一个可查询的数据库。`-j` 打印按表的合并摘要。
- **`agent audit --output-dir <dir>`（仅无状态）**——将 `-S` 运行的 HTML 报告（作为 `<dir>/vigolium-audit-report.html`）和每个驱动程序的原始 `vigolium-results/` 树的副本捆绑到一个文件夹中——一个驱动程序平铺放置，多个驱动程序在 `<dir>/<driver>/` 下命名空间。相对 `-o` 嵌套在捆绑包下；绝对路径或 `gs://` URL 转义它；`{ts}`/`{project-uuid}` 被展开一次，因此报告和原始副本共享一个目录。没有 `--output-dir` 的 `-S` 现在提示操作员原始输出保留在 `<source>/vigolium-results/` 下。
- **`vigolium server --passive-only`**——与 `-S`/`--scan-on-receive` 结合，限制对每个摄入的请求仅进行被动模块扫描（无主动扫描流量；秘密检测仍运行）。当与 `--full-native-scan-on-receive` 结合时发出警告，后者仍会爬取。

### 变更

- 示例块刷新——`replay`、`update`、`agent triage` 和 `extensions` 示例现在带有内联 `--help` 示例；`import` 示例记录了新的 SQLite 合并源。

### 内部

- 新的 `form.FillContext`（`pkg/spitolas/internal/form/fillcontext.go`，每次爬取身份内存）通过 `getValueForInput` 线程化响应感知值；`DetectedInput` 获得由 DOM 检测 JS 收集的 `DefaultValue`/`DatalistOptions`。登录喷洒位于 `crawler/login_creds.go`（每个主机单次飞行，阴性控制门控），由新的 `SpiderConfig.LoginCredentialAttempts`/`LoginCredentialFullList` 字段驱动，运行器通过 `loginCredsPolicy(intensity)` 为发现和重新爬取阶段设置这些字段。
- SQLite 文件检测集中在新的 `pkg/database/detect.go`（`IsSQLiteFile`/`HasSQLiteHeader`，基于魔术头）；`dbimport.ImportPath` 分派到新的 `ImportSQLite`（通过现有的 `database.MergeSQLiteFile`），并在导入 `Result` 上暴露 `MergeStats`。`runner_modules.go` 记录了模块切片哨兵约定（`nil`/空 → 零模块，`["all"]` → 每个模块），`--passive-only` 和 `--no-passive` 依赖于此。
- 测试覆盖新增：`form/fillcontext_test.go`、`crawler/login_creds_test.go`（+ 一个集成标记的端到端）、`database/detect_test.go`、`dbimport/importer_test.go`、`agent_audit_stateless_test.go`，以及一个 `--passive-only` 服务器选项防护。

---

## v0.1.46-beta

**2026-07-03**

一次重放人体工程学发布。`vigolium replay` 获得一个批量模式，通过变异/差异引擎重新发送每个匹配的存储记录——因此操作员（或驱动 vigolium 的编码 Agent）可以在一个命令中模糊测试整个主机的一个参数，或将所有存储的流量通过 Burp 漏斗，并输出流式 JSONL。

### 新增

- **`vigolium replay` 批量模式**——传递 `--all`（或任何 `--host`/`--method`/`--status`/`--path`/`--source`/`--search`/`--body`）会重放每个匹配的存储记录，而不是单个源，镜像 `traffic --replay` 过滤器，但每个记录都通过完整的重放引擎运行。任何 `--mutate` 应用于具有该插入点的每个记录；没有变异时，记录被逐字重新发送。结果以 JSONL 流式输出（每个记录一个稳定的 `replayOutput`），因此一个坏记录会显示内联错误而不中止批次。使用 `-c`/`--concurrency`（默认 10）节流，使用 `-n`/`--limit`（默认 100；`--all` 提升限制）限制。
- **`replay -S`/`--stateless --db <file>`**——`replay` 现在可以从独立的 `.sqlite` 或 `.jsonl` 导出中读取基线，关闭项目范围，匹配 `traffic`/`finding` 的读取语义。命令中的每个数据库接触点（记录/漏洞发现查找、`--auth-session` 合并、批量查询）都通过共享的 `openReadDB`/`effectiveProjectUUID` 助手路由。

### 变更

- `replay` 不再打印横幅——添加到横幅抑制列表，与 `scan`/`run`/`traffic` 一起，保持 JSONL 输出干净以便管道传输。
- `traffic --replay` 示例刷新——记录无状态导出重放、方法/主机过滤、以及 `--all --with-browser`。

### 内部

- 单源和批量路径通过一个共享调用形状，使用新的 `replayBatch` 函数（`cmd/internal/replay/batch.go`），该函数从 `traffic` 过滤器构建一个 SQL 查询，通过 `replayOne` 运行每个记录，并通过 `replayOutput` 结构体流式输出结果。`replayOne` 被重构以接受一个 `*replayOptions` 结构体，该结构体携带 `--mutate`、`--with-browser` 和 `--auth-session` 设置，因此单源和批量路径共享相同的重放逻辑。`cmd/internal/replay/replay.go` 中的 `runReplay` 现在根据是否存在 `--all` 或任何过滤器标志分派到 `replayBatch` 或单源路径。
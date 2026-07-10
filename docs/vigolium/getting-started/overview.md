文档API参考入门Vigolium概述复制页面Vigolium 是一款用 Go 编写的高保真 Web 漏洞扫描器。它将确定性的模块化扫描与 AI 驱动的 Agent 分析相结合，提供广泛而深入的 Web 应用程序安全问题覆盖。复制页面

Vigolium 是一款高保真 Web 漏洞扫描器，融合了 Agent AI 与原生速度、模块化和精确性。它将确定性的多阶段扫描与 AI 驱动的自主分析相结合，提供全面的安全覆盖，涵盖注入缺陷、访问控制问题、框架特定漏洞以及盲外带攻击。

该平台围绕三个关键组件构建，共同提供完整的漏洞扫描解决方案。

## 关键组件

**Vigolium CLI**：核心扫描引擎。处理所有扫描逻辑、模块执行和繁重任务，包括内容发现、基于浏览器的爬取、主动模糊测试以及 Agent AI 扫描。

**Vigolium Workbench**：一个自托管仪表板，用于可视化扫描结果、管理项目以及跟踪整个基础设施中的漏洞发现。部署在您自己的服务器上，完全控制您的数据。

**Vigolium Console**：一种基于云的解决方案，提供托管扫描、团队协作和集中报告，无需自行托管的开销。

## 扫描模式

### Agent 扫描

Agent 扫描使用 AI Agent 驱动或增强扫描过程。通过 `vigolium agent <mode>` 调用。所有 AI 调度均通过进程内的 olium 引擎路由，该引擎支持十一个供应商：`openai-codex-oauth`、`anthropic-api-key`、`anthropic-oauth`、`openai-api-key`、`openai-responses`、`anthropic-cli`、`anthropic-claude-sdk-bridge`、`anthropic-compatible`、`anthropic-vertex`、`google-vertex` 和 `openai-compatible`（Ollama / OpenRouter / LM Studio / vLLM）。

| 模式 | 命令 | 描述 |
|:-----|:-----|:-----|
| Query | `vigolium agent query` | 单次提示执行。适用于代码审查、端点发现、秘密检测。不进行网络扫描。 |
| Autopilot | `vigolium agent autopilot` | 自主 AI 驱动的渗透测试。olium 引擎自行发现端点、运行扫描并分类漏洞发现，驱动 bash、文件操作和 vigolium CLI，直到调用 `halt_scan`。支持差异聚焦运行（`--diff`）、`--intensity` 预设包和会话恢复。 |
| Swarm | `vigolium agent swarm` | AI 引导的管道，用于针对性的单请求或全范围（`--discover`）扫描。主 Agent 选择模块、生成自定义 JS 扫描器插件、运行代码审计和 SAST、执行扫描并分类结果，原生 Go 处理繁重任务，AI 在检查点介入。 |
| Audit | `vigolium agent audit` | 白盒安全审计的统一驱动调度器——运行嵌入的 `vigolium-audit` 工具、独立的 `piolium` 工具，或两者并排运行于一个父扫描下。模式：`lite` / `balanced` / `deep`。 |
| Piolium | `vigolium agent audit --driver=piolium` | Pi 原生白盒审计驱动，深度模式下最多 17 个阶段。需要单独安装 pi 运行时和 piolium 插件。 |
| Olium | `vigolium olium` / `vigolium ol` | 交互式 TUI 聊天或一次性非交互式提示，底层 Agent 运行时。 |

所有面向扫描的模式均支持 `--source` 进行源代码感知分析，并将会话工件（计划、插件、输出）存储在 `~/.vigolium/agent-sessions/` 下。请参阅设置 Agent 以进行供应商配置。

Agent 模式在无沙箱环境下运行：LLM 对主机具有完整的 shell、文件和网络访问权限，插件可以运行任意命令。请在一次性容器或 VM 中运行 Agent 模式，范围限定在您的参与范围内。开始前请参阅安全警告。

### 本地扫描

通过 `vigolium scan` 进行确定性、多阶段漏洞扫描。快速、模块化且可重复，运行内容发现、浏览器爬取、SPA 爬取以及主动/被动动态评估阶段，包含 313 个扫描器模块（198 个主动，115 个被动）。

| 类别 | 覆盖范围 |
|:-----|:-----|
| 注入 | XSS（反射型、DOM 型、SSR 水合）、SQL 注入（基于错误、布尔/时间盲注）、NoSQL 注入、SSTI/CSTI、CRLF 注入、命令注入、XXE/SAML、原型污染 |
| 访问控制 | CSRF、IDOR、授权绕过、批量赋值、禁止绕过、HTTP 方法篡改 |
| 文件与路径 | LFI、路径遍历、文件上传缺陷、目录列表、备份/敏感文件发现、路径规范化绕过 |
| API 与协议 | GraphQL 安全套件（内省、SQLi、IDOR/BOLA、DoS、批处理）、MCP 服务器安全（工具/资源/提示模糊测试、会话与来源检查）、SSRF（直接与盲注）、开放重定向、HTTP 请求走私、JWT 漏洞、JSONP 回调、WebSocket 安全、竞态条件 |
| 框架特定 | Spring Boot、Django、Laravel、Rails、Express、Next.js、Nuxt、Remix、ASP.NET/Blazor、IIS、Flask、FastAPI |
| 企业及 SaaS 平台 | Adobe Experience Manager（调度器绕过、敏感 Servlet、RCE 表面暴露、CVE 探测）、Salesforce Experience Cloud（Aura 对象/记录暴露、访客 Apex 执行、Lightning 调试模式）、ServiceNow（Widget 与 KB 数据暴露）、Microsoft Power Pages（Dataverse Web API 暴露） |
| CMS | WordPress（XML-RPC、用户枚举、AJAX 暴露）、Drupal、Joomla、CMS 安装程序暴露 |
| 云与基础设施 | Firebase（RTDB、存储、认证、函数）、云存储列表/接管、默认凭据、Web 缓存投毒、CORS 配置错误 |
| 外带 | 通过 OAST 回调的盲漏洞（盲 SSRF、盲 SSTI、OAST 探测） |

## Vigolium CLI

CLI 是 Vigolium 的核心，通过两种互补模式驱动所有扫描操作：

### CLI 亮点

- 值感知变异：按语义类型对参数值进行分类，并生成智能变异
- 多阶段管道：外部收集、内容发现、SPA 爬取和审计，由策略预设控制
- 扫描配置文件：将策略、速度、范围和模块配置打包到单个 YAML 文件中
- 多种输入格式：URL、OpenAPI/Swagger、Postman、Burp Suite、cURL、Nuclei JSONL
- 基于浏览器的爬取：Chromium 驱动的爬虫，支持 SPA、表单填充和 JS 分析
- 多会话身份验证：内联会话、会话文件或完整的身份验证配置，包含登录流程和令牌提取
- JavaScript 插件：通过嵌入式 JS 引擎实现自定义模块和钩子
- 源代码感知的 Agent 扫描：将 `--source` 与 `swarm`、`autopilot` 或 `audit` 配对，实现代码上下文感知的 AI 扫描和审计
- 并发架构：可配置的工作池，支持每主机速率限制和混合队列
- HTML 报告：自包含的 HTML 报告，包含可排序/可过滤的表格
- API 服务器模式：REST API，带有 Swagger UI、多格式数据导入、透明 HTTP 代理

## Vigolium Workbench

一个自托管的 Web 仪表板，提供用于管理和分析扫描数据的可视化界面。在您自己的基础设施上部署 Workbench，以保持对漏洞数据的完全控制，同时为团队提供直观的方式：

- 浏览和过滤扫描漏洞发现，按项目显示严重性细分
- 跟踪跨仓库和扫描历史的漏洞趋势
- 管理多个项目，支持多租户隔离
- 查看每个漏洞发现的详细请求/响应证据

## Vigolium Cloud Console

一种基于云的解决方案，适用于希望获得 Vigolium 强大功能但无需管理基础设施的团队。Console 是 Vigolium 的升级版、功能完备版本，提供托管扫描、集中报告、团队协作以及在开源核心之上添加的额外功能，让您可以专注于修复漏洞，而不是维护工具。

请访问 console.vigolium.com 查看 Cloud Console。

## 下一步

| 我想…… | 从这里开始 |
|:-----|:-----|
| 安装 CLI 并运行我的第一次扫描 | 快速入门 |
| 为任务选择合适的模式 | 选择扫描模式 |
| 运行一次性扫描（CI、临时） | 本地扫描与无状态扫描 |
| 配置 AI 供应商 | 设置 Agent |
| 了解本地扫描的工作原理 | 工作原理 |
| 选择合适的扫描策略 | 策略 |
| 深入了解各个扫描阶段 | 阶段、发现、爬取、审计、插件 |
| 尝试 Agent 扫描 | Agent 模式 |
| 让 AI 自主驱动扫描 | Autopilot |
| 运行多阶段 AI + 本地管道 | Swarm |
| 深入审计源代码 | Agent 安全审计 |
| 与 Agent 运行时聊天 | Olium |
| 将 Vigolium 作为 API 服务器运行 | 服务器模式 |
| 调整扫描设置 | 设置 |
| 导出和格式化结果 | 输出与报告 |
| 编写自定义 JS 插件 | 编写插件 |
| 浏览 REST API | API 概述 |

如有任何疑问，请随时通过 [email&#160;protected] 联系我们。

安装与快速入门：几分钟内启动并运行 Vigolium，安装、验证并运行您的第一次扫描。

下一步

⌘I

网站 Twitter GitHub Discord X LinkedIn

由 Mintlify 构建和托管，一个开发者文档平台

本页内容：关键组件、扫描模式、Agent 扫描、本地扫描、Vigolium CLI、CLI 亮点、Vigolium Workbench、Vigolium Cloud Console、下一步
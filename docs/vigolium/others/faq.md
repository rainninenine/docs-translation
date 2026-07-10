文档API 参考其他FAQ复制页面关于 Vigolium、运行模式、扫描范式、AI 供应商、数据库、多租户隔离、存储和故障排除的常见问题。复制页面最常见问题的解答。如需了解系统级概览，请从体系结构概述开始。

​通用
什么是 Vigolium？Vigolium 是一款用 Go 编写的高保真 Web 漏洞扫描器。它将确定性的基于模块的扫描与 AI 驱动的 Agent 分析相结合，提供广泛而深入的 Web 应用程序安全问题覆盖。它内置 313 个模块（198 个主动模块，115 个被动模块），涵盖注入缺陷、配置错误、信息泄露、身份验证问题等。

运行 Vigolium 有哪些方式？三种运行模式：

| 模式 | 二进制文件 | 描述 |
|------|------------|------|
| CLI 扫描器 | vigolium scan | 对目标、输入文件（OpenAPI、Postman、Burp、cURL、HAR）或源代码路径执行一次性扫描。 |
| 服务器模式 | vigolium server | 提供 REST API 服务器，包含 Swagger UI，可导入流量、触发扫描、查询漏洞发现、通过 HTTP 运行 Agent 会话。 |
| 数据导入客户端 | vigolium ingest | 轻量级客户端，用于捕获 HTTP 流量并将其转发到正在运行的服务器。 |

请参阅选择模式。

本地扫描与 Agent 扫描有何区别？本地扫描管道是完全确定性的，纯 Go 实现，无需 AI。请求流经固定的阶段序列（启发式 → 外部收集 → 爬取 → 内容发现 → 动态评估 → KnownIssueScan）。Agent 扫描使用 AI Agent 驱动或增强扫描（vigolium agent <mode>）。两者写入同一数据库，因此本地和 AI 的漏洞发现共存并一起去重。请参阅本地扫描和 Agent 扫描。

从源码构建需要什么 Go 版本？Go 1.26+。模块路径为 github.com/vigolium/vigolium。

​扫描
有哪些可用的扫描策略？策略控制哪些阶段运行以及攻击性程度：

| 策略 | 行为 |
|------|------|
| Lite | 快速表面扫描；跳过繁重的爬取和内容发现 |
| Balanced | 默认。运行所有阶段，并设置合理的限制 |
| Deep | 详尽扫描，限制更高，词表更广，并进行外部收集 |

请参阅策略。

我可以只运行扫描的某个阶段吗？可以。--only <phase> 启用单个阶段并禁用所有其他阶段；--skip <phase> 禁用特定阶段（两者互斥）。阶段别名已规范化，例如 deparos/discover → discovery，spitolas → spidering，ext → extension。vigolium run <phase> 是 scan --only <phase> 的别名。

我可以向扫描提供哪些输入格式？URL、OpenAPI/Swagger、Postman 集合、cURL 命令、Burp 原始请求、Burp XML 导出、Nuclei JSONL 输出以及 Deparos 内容发现输出。运行 vigolium --list-input-mode 可查看所有支持的格式及示例。

为什么我看到重复的漏洞发现被折叠在一起？这是设计使然。Vigolium 有三个级别的去重：请求级别（DedupManager）、内联漏洞发现级别（finding_hash 唯一约束，使用 INSERT ON CONFLICT DO NOTHING）以及阶段后分组（DeduplicateFindings() 合并共享相同模块/严重性/URL 的漏洞发现，将额外的请求/响应对折叠到幸存者的 AdditionalEvidence 中，上限为 10 个）。请参阅本地扫描 → 去重。

​Agent / AI 供应商
Agent 支持哪些 AI 供应商？所有 AI 调度都通过进程内的 olium 引擎进行。支持的供应商：

| 供应商 | 认证来源 |
|--------|----------|
| openai-codex-oauth | oauth_cred_path（来自 codex login 的 JSON） |
| anthropic-api-key | llm_api_key 或 $ANTHROPIC_API_KEY |
| anthropic-oauth | oauth_token（来自 claude setup-token）；回退到 $ANTHROPIC_API_KEY |
| openai-api-key | llm_api_key 或 $OPENAI_API_KEY |
| openai-responses | llm_api_key 或 $OPENAI_API_KEY（公共 OpenAI Responses API，/v1/responses） |
| anthropic-cli（别名 anthropic-claude-cli） | 通过 shell 调用 claude 二进制文件 |
| anthropic-claude-sdk-bridge | 通过 vigolium-audit 桥接 sidecar 登录的 Claude Code 订阅（无需密钥） |
| anthropic-compatible | custom_provider.base_url（Anthropic Messages /v1/messages 网关/代理） |
| anthropic-vertex | GCP 服务账号凭据 + 项目/位置 |
| google-vertex | 相同的 GCP 凭据；路由 gemini-* 模型 |
| openai-compatible | custom_provider.base_url（Ollama / OpenRouter / LM Studio / vLLM / …） |

在 vigolium-configs.yaml 的 agent.olium 下配置。请参阅设置 Agent。

我可以使用本地模型（Ollama、LM Studio、vLLM）吗？可以，使用 openai-compatible 供应商，它使用 OpenAI Chat Completions 线格式。将 custom_provider.base_url 设置为端点（/v1 根路径或完整的 /v1/chat/completions URL 均可），对于无需身份验证的本地服务器，将 api_key 留空。注意：线格式支持 OpenAI 风格的函数工具，但并非所有模型都支持。本地运行良好的模型：qwen2.5-coder、llama3.1 instruct、mistral-nemo。如果 Agent 从不调用工具，很可能是模型静默忽略了工具定义，请切换到经过工具训练的模型。

Query、Autopilot、Swarm 和 Audit 之间有什么区别？

- Query：单次提示；不进行网络扫描。适用于代码审查、端点发现、秘密检测。
- Autopilot：一个长时间运行的 LLM 会话，具有完整的工具访问权限，自行决定扫描内容并迭代直到停止。
- Swarm：一个多阶段管道，本地 Go 执行繁重工作，AI 在检查点进行干预（规划、分类、自定义插件生成）。
- Audit：一个前台多阶段源代码审计。vigolium agent audit 是统一的调度器，驱动嵌入的 vigolium-audit 工具和/或 piolium，通过 --driver auto|both|audit|piolium 选择。

所有面向扫描的模式都支持 --source 进行源代码感知分析。请参阅 Agent 模式。

Agent 会话存储在哪里？每次 swarm 和 autopilot 运行都会在 agent.sessions_dir（默认为 ~/.vigolium/agent-sessions/<run-uuid>/）下写入一个会话目录，包含检查点、序列化计划、会话配置、编译的插件、vigolium-audit 输出、渲染的提示以及 Agent 转录。

​数据与多租户隔离
Vigolium 使用什么数据库？基于 Bun ORM 的仓库模式，有两个可互换的后端：SQLite（默认，单二进制，零配置，本地扫描）和 PostgreSQL（共享/服务器部署，支持并发写入）。模式有意采用非规范化设计，JSONB 列承载结构化子数据，而不是单独的主机/参数表。

多租户隔离/项目范围如何工作？所有扫描数据按项目分区，没有每个项目单独的数据库。隔离是通过每个主要表上的 project_uuid 列实现的，每次读取时过滤，每次写入时标记。默认项目为 00000000-0000-0000-0000-000000000001，在 vigolium init 期间创建。选择优先级：--project-uuid > --project-name > VIGOLIUM_PROJECT_UUID > VIGOLIUM_PROJECT（旧版）> 默认。在服务器上，X-Project-UUID 标头起相同作用。请参阅数据与存储。

我升级了，现有的扫描数据会怎样？现有数据库会原地迁移。project_uuid 列会添加，默认值为默认项目的 UUID，因此多租户隔离之前的数据会归入默认项目。

云存储如何工作？存储默认禁用。启用后（storage.enabled: true），单个 S3 客户端与 GCS、S3 或自托管 MinIO 通信。项目 UUID 是桶内前缀（而非桶名），因此一个桶可以安全地容纳多个项目：gs://<project-uuid>/<key> 映射到 s3://<bucket>/<project-uuid>/<key>。gs:// URL 是一等扫描输入和导出目标。请参阅存储 API。

​服务器与 API
服务器的行为与 CLI 不同吗？不。服务器本身不拥有扫描逻辑，它包装了 CLI 使用的相同本地管道和相同 Agent 编排器，因此无论通过哪个入口点启动扫描，行为都是相同的。

如何对 API 进行身份验证？Authorization: Bearer <key>，其中密钥解析优先级为 VIGOLIUM_API_KEY 环境变量 > server.auth_api_key 配置。vigolium server -A 禁用身份验证（仅限开发环境）。元端点（/、/health、/metrics、/swagger/*）无需身份验证。

为什么我的扫描/Agent 请求返回 202 而不是结果？扫描和 Agent 运行是长时间运行的，因此 API 采用启动并轮询模式。运行端点返回 202 Accepted 并附带 UUID（如果已有一个正在运行，则返回 409 Conflict，一次只能运行一个 Agent 会话）。轮询状态端点直到其离开 running 状态，然后获取工件。选择 "stream": true 可使用服务器发送事件。

我可以在不检测工具的情况下捕获流量吗？可以。vigolium server --ingest-proxy-port 9003 打开一个记录 HTTP 代理，通过它路由的纯 HTTP 流量会被捕获到数据库中（HTTPS CONNECT 隧道会透传而不记录）。将 curl -x、httpx -proxy、nuclei -proxy 等指向它。请参阅服务器与 API。

​故障排除
Agent 从不调用工具 / 仅以散文形式回复几乎总是模型问题。OpenAI 风格的函数工具是线格式的一部分，但并非所有模型都遵守，较小的本地模型通常忽略工具定义。请切换到经过工具训练的模型（qwen2.5-coder、llama3.1 instruct、mistral-nemo 或托管的 Claude/GPT 模型）。

SQLite 并发/忙超时设置似乎不适用默认的 modernc SQLite 驱动程序需要以 _pragma=name(value) 形式设置 pragma。mattn 风格的 _busy_timeout= DSN 参数会被静默忽略。调整并发写入行为时，请相应调整 DSN。

主机在扫描中途停止扫描Vigolium 具有每主机断路器（hosterrors.Cache）。在连续出现 MaxHostError 次错误（默认 30 次）后，该主机被隔离，执行器跳过其后续项目。成功响应会重置计数器（除非已达到阈值）。这可以保护扫描吞吐量免受无响应主机的影响。

扫描似乎按主机限速这是设计使然。HostRateLimiter 为每个主机提供一个带缓冲通道的信号量，容量为 max-per-host（CLI 默认每主机 30 个并发）。通过配置或 CLI 并发标志调整扫描速度。

一个嘈杂的模块只产生少量漏洞发现然后停止这是预期的。执行器强制执行每模块漏洞发现上限（MaxFindingsPerModule，默认 10），一旦模块发出那么多漏洞发现，后续结果将被抑制，并记录一次性警告。这可以防止注入式模块淹没输出。

我在哪里可以看到所有可用命令和示例？vigolium --help 列出所有命令和全局标志；vigolium <command> --help 查看特定命令；vigolium --full-example 查看精选导览。vigolium doctor 诊断配置和工具就绪状态。请参阅 CLI 参考。

仍有疑问？请联系 [email protected]。上一页更新日志Vigolium 的发布说明和版本历史⌘IwebsitetwittergithubdiscordxlinkedinPowered by本文档基于 Mintlify 构建和托管，Mintlify 是一个开发者文档平台本页内容通用扫描Agent / AI 供应商数据与多租户隔离服务器与 API故障排除
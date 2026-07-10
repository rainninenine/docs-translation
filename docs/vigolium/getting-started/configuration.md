文档API参考入门指南设置参考复制页面Vigolium 使用分层配置系统，从多个来源合并设置。本文档涵盖配置文件格式、环境变量以及每个可配置部分。复制页面​配置文件位置
主配置文件位于 `~/.vigolium/vigolium-configs.yaml`。首次运行时，系统会自动创建该文件并包含合理的默认值。
Vigolium 按以下顺序搜索配置：

通过 `--config` 标志指定的路径（如果未找到则报错）
`~/.vigolium/vigolium-configs.yaml`
`./vigolium-configs.yaml`（当前工作目录）

如果未找到配置文件，则使用内置默认值。
​配置优先级
设置按从高到低的优先级解析：

CLI 标志 - 例如 `--concurrency 100`、`--rate-limit 50`
环境变量 - 例如 `VIGOLIUM_API_KEY`、`VIGOLIUM_PROJECT`
扫描配置文件 - 通过 `--scanning-profile <name>` 加载（来自 `~/.vigolium/profiles/`）
项目级配置 - 每个项目的覆盖文件位于 `~/.vigolium/projects/<uuid>/config.yaml`
主配置文件 - `~/.vigolium/vigolium-configs.yaml`
内置默认值 - 硬编码在 Go 源代码中

优先级高的来源会覆盖优先级低的来源。在配置文件中，可以使用 `${VAR}` 或 `$VAR` 语法引用环境变量，并在加载时展开。
​环境变量
变量用途`VIGOLIUM_API_KEY`REST 服务器和数据导入客户端身份验证的 API 密钥`VIGOLIUM_PROJECT`CLI 操作的默认项目 UUID（相当于 `--project`）`VIGOLIUM_PROXY`HTTP/SOCKS 代理 URL，当未设置 `--proxy` 时使用`VIGOLIUM_HOME`Vigolium 数据的基础目录（由安装程序使用；默认为 `~/.vigolium`）
任何环境变量也可以在 `vigolium-configs.yaml` 内部进行插值：
database:
  postgres:
    password: ${VIGOLIUM_DB_PASSWORD}

​CLI 配置覆盖
使用 `vigolium config set` 通过点符号键更新单个配置值：
vigolium config set scanning_pace.concurrency 100
vigolium config set database.driver postgres
vigolium config set notify.enabled true
vigolium config set notify.severities high,critical
vigolium config set server.service_port 8080

这些命令直接修改主配置文件。对于扫描过程中的一次性覆盖，请改用 CLI 标志。
​配置部分
​scanning_strategy
控制每个策略预设运行哪些扫描阶段。
scanning_strategy:
  default_strategy: balanced    # lite | balanced | deep
  heuristics_check: basic
  scanning_profile: &quot;&quot;          # 要自动加载的配置文件名称
  profiles_dir: ~/.vigolium/profiles/

  session:
    session_dir: ~/.vigolium/sessions/
    use_in_discovery: true       # 在内容发现/爬取期间应用会话头
    compare_enabled: true        # 在动态评估中跨会话重放 IDOR/BOLA
    reauth_interval: &quot;&quot;          # 例如 &quot;15m&quot; 定期刷新令牌
    reauth_on_status: []         # 例如 [401, 403]
    validate_url: &quot;&quot;             # 登录后用于验证凭据的 GET URL

  # 每个策略的阶段开关（规范名称）：
  balanced:
    discovery: true
    spidering: true
    known_issue_scan: true
    dynamic-assessment: true
    external_harvesting: false

可用策略及其默认阶段：
阶段litebalanceddeepexternal_harvesting--yesdiscovery-yesyesspidering-yesyesknown_issue_scan-yesyesdynamic-assessmentyesyesyes

阶段别名：`dynamic-assessment` 是主动/被动漏洞扫描的规范名称。`audit`、`dast` 和 `assessment` 是 `--only` / `--skip` 标志接受的别名。`discovery` 接受 `deparos` / `discover`；`spidering` 接受 `spitolas`；`extension` 接受 `ext`。

对于源代码感知的白盒分析，请使用 `vigolium agent swarm --source <path>` 或 `vigolium agent audit --source <path>` 而不是本地扫描策略。请参阅 Agent 模式。
​scanning_pace
集中式速度控制。常用值作为基线；每个阶段的子部分会覆盖它们。
scanning_pace:
  concurrency: 50          # 全局工作线程数
  rate_limit: 100          # 所有主机的最大请求数/秒
  max_per_host: 10         # 每个主机的最大并发请求数
  max_duration: 2h         # 扫描阶段的全局时间上限

  # 每个阶段的覆盖（0 = 继承自公共设置）：
  discovery:
    concurrency: 0
    rate_limit: 0
    concurrency_factor: 0  # 公共并发数的乘数
    duration_factor: 0     # 公共最大持续时间的乘数
  spidering:
    duration_factor: 0.15  # 例如 2h * 0.15 = 18m
  known_issue_scan:
    duration_factor: 3.0
  external_harvester:
    duration_factor: 0.2
  audit:
    duration_factor: 1.0
    parallel_passive: true         # 并行运行被动模块
    feedback_drain_timeout: 500ms  # 等待反馈循环项

​discovery
内容发现（目录/文件暴力破解）。
discovery:
  mode: files_and_dirs         # files_and_dirs | files_only | dirs_only
  scope_mode: subdomain        # any | subdomain | exact
  save_response_body: true
  enable_malformed_path_probe: false
  dedup_cluster_cap: 10        # 每个集群最多保留 N 个几乎相同的响应（省略 = 10，0 = 禁用）
  auto_fuzz_low_yield: true    # 当爬取结果为空或遇到 SSO 墙时，自动对原始目标启用 FUZZ（省略 = 开启）
  enrich_targets: false        # 将爬取/收集的路径馈送到内容发现中

  recursion:
    enabled: true
    max_depth: 5

  wordlists:
    short_file_path: &quot;&quot;        # 自定义字典路径
    long_file_path: &quot;&quot;
    short_dir_path: &quot;&quot;
    long_dir_path: &quot;&quot;
    fuzz_wordlist_path: &quot;&quot;
    use_observed_names: true
    use_observed_paths: true
    use_observed_files: true
    enable_numeric_fuzzing: false

  extensions:
    test_custom: true
    custom_list: []
    test_observed: true
    test_backup_extensions: true
    backup_extensions: []
    test_no_extension: true

  engine:
    case_sensitivity: auto_detect  # auto_detect | sensitive | insensitive
    timeout: 10s                   # 每个请求的超时时间（1s-300s）
    custom_headers: {}
    enable_cookie_jar: false
    max_consecutive_errors: 0
    max_consecutive_waf_blocks: 0
    observed_max_items: 4000
    disable_kingfisher: false

​spidering
基于浏览器的爬取。
spidering:
  max_depth: 0               # 0 = 无限制
  max_states: 0              # 0 = 无限制
  max_duration: 30m
  max_consecutive_fails: 100
  headless: true
  browser_count: 1
  strategy: adaptive         # normal | random | oldest_first | shallow_first | adaptive
  include_response_body: true
  browser_engine: chromium   # chromium | ungoogled | fingerprint
  no_cdp: false              # 禁用 CDP 事件监听器检测
  no_forms: false            # 禁用自动表单填充

  # AI 驾驶模式（Agent 控制的浏览器）：
  pilot_mode: false
  pilot_auto_register: true
  pilot_username: &quot;&quot;
  pilot_password: &quot;&quot;
  pilot_screenshot: true
  pilot_max_retries: 2
  pilot_stall_timeout: 7m

​dynamic-assessment
控制哪些扫描器模块运行以及 JavaScript 插件设置。（原为 audit。）
dynamic-assessment:
  max_feedback_rounds: 1            # 新发现 URL 的重新扫描轮次
  max_findings_per_module: 15       # 0 = 无限制

  enabled_modules:
    active_modules: [&quot;all&quot;]         # [&quot;all&quot;] 或模块 ID 列表
    passive_modules: [&quot;all&quot;]

  extensions:
    enabled: false
    extension_dir: ~/.vigolium/extensions/
    custom_dir: []                  # 额外的脚本路径
    variables: {}                   # 传递给脚本的键值对
    limits:
      timeout: 30s
      max_memory_mb: 128

​scope
定义扫描范围。排除规则优先于包含规则。
scope:
  applied_on_ingest: false       # 在数据导入期间强制执行范围（不仅仅是扫描）
  cli_origin_mode: relaxed       # relaxed | all | balanced | strict
  ignore_static_file: true       # 跳过图片、字体、视频、音频等
  max_request_body_size: 1048576     # 1 MB
  max_response_body_size: 524288000  # 500 MB
  body_size_exceeded_action: truncate  # truncate | drop | skip-scan

  host:
    include: [&quot;*&quot;]
    exclude: []
  path:
    include: [&quot;*&quot;]
    exclude: []
  status_code:
    include: [&quot;*&quot;]
    exclude: []
  request_content_type:
    include: [&quot;*&quot;]
    exclude: []
  response_content_type:
    include: [&quot;*&quot;]
    exclude: []
  request_string:
    include: []
    exclude: []
  response_string:
    include: []
    exclude: []

​server
REST API 服务器设置。
server:
  auth_api_key: &quot;&quot;                 # 如果为空则自动生成；也可通过 VIGOLIUM_API_KEY 设置
  service_port: 9002
  ingest_proxy_port: 0             # 0 = 禁用
  mirror_fs_path: &quot;&quot;               # 将导入的流量和漏洞发现镜像到此目录，作为实时文件系统树（参见 --mirror-fs）
  cors_allowed_origins: reflect-origin
  enable_metrics: true
  agent_heavy_max: 5               # 通过 API 的最大并发 autopilot/swarm 运行数
  agent_light_max: 10              # 通过 API 的最大并发 query/chat 运行数
  agent_queue_timeout: 30s         # 在返回 429 之前等待 Agent 槽位的时间
  license: &quot;&quot;                      # 可选的许可证标签，在 /server-info 中显示

​agent
AI Agent 集成。每次 Agent 调用都通过进程内的 olium 运行时调度，没有子进程 SDK 或 ACP 后端。
agent:
  default_agent: olium
  templates_dir: ~/.vigolium/prompts/
  sessions_dir: ~/.vigolium/agent-sessions/
  stream: true                     # 实时输出流

  # Olium 引擎 — 由每个 Agent 子命令使用
  olium:
    provider: openai-compatible    # openai-codex-oauth | anthropic-api-key | anthropic-oauth | openai-api-key | openai-responses | anthropic-cli | anthropic-claude-sdk-bridge | anthropic-compatible | anthropic-vertex | google-vertex | openai-compatible (默认: openai-compatible)
    model: gemma4:latest           # 空 = 供应商默认；匹配 custom_provider.model_id 默认值（openai-compatible / anthropic-compatible）
    oauth_cred_path: ~/.codex/auth.json    # openai-codex-oauth 供应商
    bridge_binary: &quot;&quot;              # anthropic-claude-sdk-bridge：托管 SDK 桥接的 vigolium-audit 二进制文件路径（空 = 嵌入式 blob，然后是 PATH）
    oauth_token: &quot;&quot;                # anthropic-oauth 的 Bearer 令牌（来自 `claude setup-token`）；支持 ${ENV_VAR}
    llm_api_key: &quot;&quot;                # 用于 anthropic-api-key / openai-api-key / openai-responses；支持 ${ENV_VAR}；回退到供应商环境变量
    reasoning_effort: medium       # minimal | low | medium | high | xhigh (codex)
    system_prompt: &quot;&quot;              # 覆盖内置的 olium 系统提示
    max_tokens: 1000000
    temperature: 0.0
    max_turns: 32                  # 工具循环迭代上限
    cache_size: 1024               # LRU 条目数；0 禁用
    max_concurrent: 4              # 同时进行中的供应商调用的全局上限
    call_timeout_sec: 600          # 每次调用的截止时间；-1 = 无强制超时

    # 无论规划器选择如何，autopilot/swarm 始终加载的技能。
    # 空 = 内置默认值 [triage-finding, write-jsext]。CLI 标志
    # --skill / --skill-tag / --no-skill-filter 每次运行覆盖选择。
    always_on_skills: []

    # 自定义后端：当 provider == openai-compatible（OpenAI Chat
    # Completions — Ollama、OpenRouter、LM Studio、vLLM、Together、Groq 等）或
    # provider == anthropic-compatible（Anthropic Messages /v1/messages 网关）时使用。
    custom_provider:
      base_url: http://localhost:11434/v1   # 例如 http://localhost:11434/v1 (Ollama)
      model_id: gemma4:latest               # 后端特定的模型 ID
      api_key: &quot;&quot;                           # 可选；留空以跳过 Authorization 头；支持 ${ENV_VAR}
      extra_headers: []                     # curl 风格的 &quot;Key: Value&quot; 字符串列表，在标准头之后应用

      # OpenRouter 供应商路由 — 请求 &quot;provider&quot; 对象的类型化旋钮。
      # 仅设置您设置的字段；未设置的字段将从请求体中删除。
      provider_routing:
        order: []                  # 按偏好顺序的供应商 slug
        only: []                   # 限制为这些供应商 slug
        ignore: []                 # 排除这些供应商 slug
        allow_fallbacks: true      # false = 严格（仅限所选供应商）
        sort: &quot;&quot;                   # price | throughput | latency
        quantizations: []          # 例如 [fp8, int8]
        data_collection: &quot;&quot;        # allow | deny
        require_parameters: false  # 仅限遵守每个请求参数的供应商
        zdr: false                 # 仅限零数据保留供应商

      # 通用 JSON 请求体透传，合并到每个 openai-compatible 请求中
      # （OpenRouter 扩展、vLLM/Together 请求体选项等）。保留键
      # （model、messages、tools、stream、stream_options）将被拒绝。如果您也使用
      # provider_routing，请不要在此处设置 `provider` 键 — 使用其中一个。
      extra_body: {}

    # Vertex AI（anthropic-vertex / google-vertex）— GCP 项目/区域。
    # 凭据来自 `oauth_cred_path`（服务账户 JSON）或
    # $GOOGLE_APPLICATION_CREDENTIALS。$GOOGLE_CLOUD_PROJECT 和
    # $GOOGLE_CLOUD_LOCATION 会覆盖这些 YAML 值。
    google_cloud_project: &quot;&quot;       # GCP 项目 ID
    google_cloud_location: &quot;&quot;      # GCP 区域；默认为 us-central1

  # JavaScript 插件 Agent API（扩展中的 vigolium.agent.*）的 LLM 配置
  llm:
    provider: anthropic            # anthropic | openai
    model: claude-sonnet-4-20250514
    api_key: &quot;&quot;                    # 内联密钥（优先使用 api_key_env）
    api_key_env: &quot;&quot;                # 环境变量名称（默认：ANTHROPIC_API_KEY 或 OPENAI_API_KEY）
    base_url: &quot;&quot;                   # 用于 OpenAI 兼容供应商的自定义端点
    max_tokens: 4096
    temperature: 0.0
    cache_size: 256                # LRU 条目数（0 = 禁用）
    cache_ttl: 300                 # 秒

  # swarm/autopilot 的数据库上下文丰富限制
  context_limits:
    max_findings: 50
    max_endpoints: 100
    max_high_risk: 20
    min_risk_score: 50

  # Autopilot 护栏（SDK 时代；在 olium 下主要为信息性）
  guardrails:
    log_commands: false
    max_turns: 0                   # 0 = 自动（MaxCommands * 3）
    disallowed_tools: []

  # 可选的 Agent 浏览器集成（Bash 工具可以驱动真实的 Chromium 用于需要身份验证的流程）
  browser:
    enable: true
    binary_path: agent-browser     # 默认：在 $PATH 中查找

  # Vigolium-audit 集成（嵌入式工具）
  audit:
    enable: false                  # 由 autopilot/swarm 上的 --audit 标志自动启用
    mode: lite                     # lite | balanced | deep | mock
    sync_interval: 30              # 状态同步之间的秒数
    # audit 驱动程序运行 `claude` 或 `codex` CLI。此处没有
    # 平台旋钮 — Agent 从 agent.olium.provider 解析
    # （anthropic-* → claude，openai-* → codex），每次运行可通过
    # `vigolium agent audit` 的 --provider 或 --agent {claude|codex} 覆盖。

供应商快速参考：
供应商默认模型凭据来源openai-codex-oauthgpt-5.5oauth_cred_path (~/.codex/auth.json)anthropic-api-keyclaude-opus-4-7llm_api_key 或 $ANTHROPIC_API_KEYanthropic-oauthclaude-opus-4-7oauth_token（来自 claude setup-token）；回退到 $ANTHROPIC_API_KEYopenai-api-keygpt-5.5llm_api_key 或 $OPENAI_API_KEYopenai-responsesgpt-5.5llm_api_key 或 $OPENAI_API_KEY（公共 OpenAI Responses API，/v1/responses）anthropic-cli（别名 anthropic-claude-cli）claude-opus-4-7本地 $PATH 上的 claude 二进制文件anthropic-claude-sdk-bridgeClaude Code 默认通过 vigolium-audit 桥接边车登录的 Claude Code 订阅（无需密钥）；bridge_binary / --bridge-bin 覆盖anthropic-compatible通过 custom_provider.model_idagent.olium.custom_provider（Anthropic Messages /v1/messages 网关或代理）anthropic-vertexclaude-opus-4-6通过 google_cloud_project / google_cloud_location 的 GCP 服务账户 JSON（或 ADC）google-vertexgemini-2.5-pro通过 google_cloud_project / google_cloud_location 的 GCP 服务账户 JSON（或 ADC）openai-compatible（默认）gemma4:latest（通过 custom_provider.model_id）agent.olium.custom_provider（Ollama、OpenRouter、LM Studio、vLLM 等）
CLI 标志 --provider、--model、--oauth-cred、--oauth-token、--llm-api-key、--base-url、--bridge-bin 每次调用会覆盖这些设置。REST API 不会镜像这些覆盖，服务器端工作负载仅使用 YAML 配置。请参阅设置 Agent 获取逐步指南，或参阅 Olium Agent 获取完整的供应商详细信息。
​database
存储后端。默认使用 SQLite；对于多用户部署支持 PostgreSQL。
database:
  enabled: true
  driver: sqlite                   # sqlite | postgres

  sqlite:
    path: ~/.vigolium/database-vgnm.sqlite
    busy_timeout: 15000
    journal_mode: WAL              # DELETE | TRUNCATE | PERSIST | MEMORY | WAL | OFF
    synchronous: NORMAL            # OFF | NORMAL | FULL | EXTRA
    cache_size: 10000
    max_open_conns: 8              # WAL 读取器池：并发读取器 + 1 个写入器

  postgres:
    host: localhost
    port: 5432
    user: vigolium
    password: &quot;&quot;
    database: vigolium
    sslmode: disable
    max_open_conns: 25
    max_idle_conns: 5
    conn_max_lifetime: 5m

​known_issue_scan
由 Nuclei 模板引擎驱动的已知问题扫描。
known_issue_scan:
  tags: []                         # nuclei 模板标签（空 = 全部）
  exclude_tags: [dos]
  severities: []                   # 过滤：critical、high、medium、low、info
  templates_dir: &quot;&quot;                # 自定义模板路径
  enrich_targets: true             # 将发现的路径馈送到已知问题扫描中

  # 重新映射漏洞发现记录的严重性，键为 nuclei 模板 ID。
  # 默认降低通用 config.json 暴露的严重性（通常仅为公共基础
  # URL / 功能标志）。添加您自己的模板 ID → 严重性条目。
  severity_overrides:
    config-json-exposure-fuzz: medium   # critical | high | medium | low | info

​mutation_strategy
控制主动扫描期间参数值的变异方式。
mutation_strategy:
  default_modes: [append]

  value_aware:
    enabled: true
    max_per_intent: 5
    default_intents: [neighbor, boundary, escalation]
    enum_mappings: {}              # 自定义枚举升级对
    param_synonyms: {}             # 自定义参数名称同义词

  field_type_defaults:
    email: [&quot;[email&#160;protected]&quot;, &quot;[email&#160;protected]&quot;]
    uuid: [&quot;550e8400-e29b-41d4-a716-446655440000&quot;]
    integer: [&quot;1&quot;, &quot;100&quot;, &quot;999&quot;]
    # ...（所有标准类型都有内置默认值）

​external_harvester
扫描前从公共数据源收集情报。
external_harvester:
  sources: [wayback, commoncrawl, alienvault]
  # 其他来源：urlscan、virustotal（需要 API 密钥）

  api_keys:
    urlscan: &quot;&quot;
    virustotal: &quot;&quot;

​oast
通过 interactsh 回调进行带外应用安全测试。
oast:
  enabled: true
  server_url: oast.pro
  token: &quot;&quot;                        # 可选的身份验证令牌
  poll_interval: 5                 # 秒
  grace_period: 10                 # 扫描后等待延迟回调的秒数
  oast_url: &quot;&quot;                     # 固定的回调 URL（空 = 自动生成）
  blind_xss_src: &quot;&quot;                # 用于盲 XSS 载荷的 JS 脚本 src
  enabled_blind_xss: false

​source_aware
克隆的源代码仓库的存储位置。当 `--source` 接收 git URL 时使用（autopilot、swarm、audit、query）。静态分析工具（ast-grep、semgrep 等）已被移除，对于 AI 驱动的代码审计，请使用 `vigolium agent audit` 或 `vigolium agent swarm --source --code-audit`。
source_aware:
  storage_path: ~/.vigolium/source-aware/   # 克隆仓库的基础目录
  clone_depth: 1                            # `git clone --depth`（1 = 浅克隆）

​storage
用于源代码上传/下载和扫描结果归档的云存储集成。使用 S3 兼容 API，支持 GCS（通过 HMAC）、AWS S3 和 MinIO。
storage:
  enabled: false
  driver: gcs                  # gcs | s3 | minio
  endpoint: &quot;&quot;                 # gcs/s3 自动；minio 必需
  bucket: ${VIGOLIUM_STORAGE_BUCKET_NAME}
  region: asia-southeast1
  access_key: ${VIGOLIUM_STORAGE_ACCESS_KEY}
  secret_key: ${VIGOLIUM_STORAGE_SECRET_KEY}
  use_ssl: true
  path_style: false            # 某些 MinIO 部署需要

启用后，使用 `--upload-results` 调用的 Agent 运行会将其会话包归档到 `<bucket>/<project-uuid>/agentic-scans/<run-uuid>/results.tar.gz`。本地扫描使用 `<bucket>/<project-uuid>/native-scans/<scan-uuid>/results.tar.gz`。请参阅存储 API 获取上传/下载端点。
​notify
通过 Telegram 或 Discord 实时通知漏洞发现。
notify:
  enabled: false
  severities: [high, critical, medium]

  telegram:
    bot_token: &quot;&quot;
    chat_id: &quot;&quot;

  discord:
    webhook_url: &quot;&quot;

​扫描配置文件
扫描配置文件是存储在 `~/.vigolium/profiles/` 中的 YAML 文件，用于覆盖主配置的子集。它们可以调整以下任意组合：`scanning_strategy`、`scanning_pace`、`discovery`、`spidering`、`known_issue_scan`、`audit`、`external_harvester`、`mutation_strategy` 和 `scope`。
使用以下命令应用配置文件：
vigolium scan --scanning-profile aggressive

这将加载 `~/.vigolium/profiles/aggressive.yaml` 并将其覆盖到活动配置上。配置文件中只有非零字段会覆盖基础配置；未指定的字段保持不变。
内置配置文件捆绑在 `public/presets/profiles/` 中。有关详细信息，请参阅 native-scan/scanning-modes-overview。
​项目级配置
每个项目可以在 `~/.vigolium/projects/<uuid>/config.yaml` 拥有自己的配置覆盖。这使用与扫描配置文件相同的格式，并在项目激活时自动应用。
使用以下命令管理项目配置：
vigolium project config set scanning_pace.concurrency 200
vigolium project config show

有关完整的项目管理文档，请参阅 projects。上一页Vigolium 备忘录一份单页、可复制粘贴的参考，涵盖您最常用的工作流程：实时流量镜像、通过 Burp 重放、转发流量的被动/秘密扫描、导入外部数据、并行扇出、恢复、规范驱动扫描、单请求扫描、内容发现、过滤到单一漏洞类别或技术、浏览数据库中记录的流量、使用编码 Agent 分类漏洞发现、重现漏洞利用、设置 AI Agent 以及检查或编辑配置。下一页⌘IwebsitetwittergithubdiscordxlinkedinPowered by本文档使用 Mintlify 构建和托管，Mintlify 是一个开发者文档平台本页内容配置文件位置配置优先级环境变量CLI 配置覆盖配置部分scanning_strategyscanning_pacediscoveryspideringdynamic-assessmentscopeserveragentdatabaseknown_issue_scanmutation_strategyexternal_harvesteroastsource_awarestoragenotify扫描配置文件项目级配置
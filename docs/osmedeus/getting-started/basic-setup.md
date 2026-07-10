# 基本设置

> Osmedeus 设置管理与配置

!!! note "Osmedeus 设置通过 YAML 文件管理，默认存储在 `~/osmedeus-base/osm-settings.yaml`。"

您可以直接修改配置文件，或使用设置 CLI。

## 1. 直接编辑 YAML 文件

![描述性替代文本](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/cli/cli-config.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=07609766ae7d20e58e9b287830b058c8)

请参阅下方带注释的完整 YAML 设置，了解每个字段的含义。

要提高扫描质量，您通常需要修改 `global_vars.[keyname]` 字段，该字段保存可在工作流中使用的全局变量。

一些示例键包括 `GITHUB_API_KEY, SHODAN_API_KEY, PASSIVETOTAL_API_KEY` 等。

<details>
<summary>带注释的完整 YAML 设置</summary>

  ```yaml
  # Osmedeus 配置文件
  # 此文件包含 osmedeus 的所有可用配置选项。
  # 将此文件复制到 ~/osmedeus-base/osm-settings.yaml 并根据需要自定义。

  # =============================================================================
  # 基础文件夹
  # =============================================================================
  # 所有 osmedeus 数据的根目录（工作流、二进制文件、数据等）
  # 环境变量如 $HOME 会自动展开
  base_folder: $HOME/osmedeus-base

  # =============================================================================
  # 环境路径
  # =============================================================================
  # 各种 osmedeus 组件的目录路径
  # 使用 {{base_folder}} 引用上面的 base_folder 值
  environments:
    # 二进制可执行文件路径（如 nmap、ffuf 等工具）
    external_binaries_path: "{{base_folder}}/external-binaries"

    # 数据目录，用于存储资产、字典等
    external_data: "{{base_folder}}/external-data"

    # 外部配置文件（如 nuclei 模板等）
    external_configs: "{{base_folder}}/external-configs"

    # 扫描项目空间的输出目录
    # 每个目标在此处拥有自己的子目录
    workspaces: $HOME/workspaces-osmedeus

    # 包含工作流 YAML 文件的目录
    # 子目录：flows/、modules/
    workflows: "{{base_folder}}/workflows"

    # 项目空间快照目录（zip 归档）
    # 由 snapshot-download API 端点使用
    snapshot: "{{base_folder}}/snapshot"

    # Markdown 报告模板目录
    # 由 render_markdown_report() 函数使用
    markdown_report_templates: "{{base_folder}}/markdown-report-templates"

    # 外部代理配置目录
    # 用于 LLM Agent 命令、技能及相关配置
    external_agent_configs: "{{base_folder}}/external-agent-configs"

  # =============================================================================
  # 数据库配置
  # =============================================================================
  # Osmedeus 支持 SQLite（默认）和 PostgreSQL
  database:
    # 数据库引擎："sqlite" 或 "postgresql"
    db_engine: sqlite

    # SQLite：数据库文件路径
    # 使用 PostgreSQL 时忽略
    db_path: "{{base_folder}}/database-osm.sqlite"

    # PostgreSQL 连接设置
    # 仅当 db_engine 为 "postgresql" 时使用
    host: localhost
    port: 5432
    username: osmedeus
    password: osmedeus
    db_name: osmedeus

    # 连接超时时间（秒）
    connection_timeout: 60

    # PostgreSQL SSL 模式：disable、require、verify-ca、verify-full
    ssl_mode: disable

  # =============================================================================
  # 服务器配置
  # =============================================================================
  # Web 界面的 REST API 服务器设置
  server:
    # 服务器绑定的主机
    # 使用 "0.0.0.0" 监听所有接口
    # 使用 "127.0.0.1" 仅监听本地主机
    host: "0.0.0.0"

    # API 服务器端口号
    port: 8002

    # 静态 UI 文件路径
    # 默认：{{base_folder}}/ui/ - 如果此目录存在，将在 /ui 下提供
    # 设置为空字符串以禁用 UI 服务
    ui_path: "{{base_folder}}/ui/"

    # 项目空间静态文件的随机前缀（如果为空则自动生成 16 个字符）
    # 用作直接访问 workspaces 文件夹的 URL 路径段
    workspace_prefix_key: ""

    # 身份验证凭据（用户名:密码映射）
    # 支持多个用户
    simple_user_map_key:
      osmedeus: osmedeus-admin

    # JWT（JSON Web Token）设置
    jwt:
      # 用于签名 JWT 令牌的密钥
      # 重要：在生产环境中使用强唯一密钥！
      secret_signing_key: change-this-secret-in-production

      # 令牌过期时间（分钟）
      expiration_minutes: 180

    # 在 HTTP Server 标头和 /server-info 端点中显示的许可证类型
    license: "open-source"

  # =============================================================================
  # 扫描策略配置
  # =============================================================================
  # 不同扫描强度级别的线程数
  # 值越高 = 更快但更激进的扫描
  # 值越低 = 更慢但对目标系统更温和
  scan_tactic:
    # 激进/快速模式 - 最大并行度
    # 使用：osmedeus scan -t target --tactic aggressive
    aggressive: 40

    # 默认/正常模式 - 平衡方法
    # 未指定策略时使用
    default: 10

    # 温和/彻底模式 - 最小并行度
    # 使用：osmedeus scan -t target --tactic gently
    gently: 5

  # =============================================================================
  # Redis 配置（可选）
  # =============================================================================
  # 分布式扫描模式需要 Redis
  # 将 host 留空以禁用 Redis
  redis:
    # Redis 服务器主机名
    # 留空以禁用分布式模式
    host: ""

    # Redis 服务器端口
    port: 6379

    # Redis 身份验证（如果需要）
    username: ""
    password: ""

    # Redis 数据库编号（0-15）
    db: 0

    # 连接超时时间（秒）
    connection_timeout: 60

  # =============================================================================
  # 全局变量
  # =============================================================================
  # 用户定义的变量，可在工作流中通过 {{VARIABLE_NAME}} 使用
  # 变量可以选择导出为环境变量
  # 使用 _API_KEY 后缀表示敏感值
  #
  # 格式：
  #   VARIABLE_NAME:
  #     value: "the-value"
  #     as_env: true  # 可选：导出为环境变量（默认：true）
  #
  # 在工作流中的使用示例：
  #   - bash: "echo {{GITHUB_API_KEY}}"
  #   - bash: "shodan search $SHODAN_API_KEY"  # 使用环境变量
  global_vars:
    # GitHub 个人访问令牌，用于 API 访问
    GITHUB_API_KEY:
      value: ""
      as_env: true  # 导出为 GITHUB_API_KEY

    # Shodan API 密钥，用于被动侦察
    SHODAN_API_KEY:
      value: ""
      as_env: true  # 导出为 SHODAN_API_KEY

    # Censys API 密钥，用于证书/主机搜索
    CENSYS_API_KEY:
      value: ""
      as_env: true  # 导出为 CENSYS_API_KEY

    # PassiveTotal API 密钥，用于被动 DNS/WHOIS
    PASSIVETOTAL_API_KEY:
      value: ""
      as_env: true  # 导出为 PASSIVETOTAL_API_KEY

    # 根据需要添加更多 API 密钥（使用 _API_KEY 后缀表示密钥）

  # =============================================================================
  # 通知配置
  # =============================================================================
  # 扫描完成或发现有趣结果时发送通知
  notification:
    # 通知供应商："telegram"（未来：slack、discord、webhook）
    provider: telegram

    # 主开关，启用/禁用所有通知
    enabled: false

    # Telegram 机器人设置
    # 通过 @BotFather 创建机器人并获取令牌
    # 通过向 @userinfobot 发送消息获取您的聊天 ID
    telegram:
      # 来自 @BotFather 的机器人令牌
      bot_token: ""

      # 接收消息的聊天 ID（可以是用户或群组）
      chat_id: 0

      # 启用 Telegram 通知
      enabled: false

  # =============================================================================
  # 云存储配置（可选）
  # =============================================================================
  # 用于备份扫描结果的 S3 兼容存储
  # 支持 AWS S3、MinIO、Google Cloud Storage、DigitalOcean Spaces 等
  storage:
    # 存储供应商："s3"、"minio"、"gcs"、"spaces" 等
    provider: s3

    # 存储端点 URL
    # AWS S3：留空或使用区域特定端点
    # MinIO："http://localhost:9000"
    # DigitalOcean："https://nyc3.digitaloceanspaces.com"
    endpoint: ""

    # 访问凭据
    access_key_id: ""
    secret_access_key: ""

    # 用于存储结果的存储桶名称
    bucket: ""

    # 云区域（例如 us-east-1、eu-west-1）
    region: us-east-1

    # 使用 SSL/TLS 连接
    use_ssl: true

    # 启用云存储上传
    enabled: false

  # =============================================================================
  # LLM 配置（可选）
  # =============================================================================
  # 用于 AI 驱动功能的大型语言模型设置
  # 支持 Ollama、OpenAI、Anthropic 等供应商
  # 可配置多个供应商，在出错或达到速率限制时自动轮换
  llm_config:
    # LLM 供应商列表（出错/速率限制时轮换到下一个）
    llm_providers:
      # 主要供应商（首先使用）
      - provider: ollama
        base_url: "http://localhost:11434/v1/chat/completions"
        auth_token: ""
        model: "gpt-oss:120b-cloud"
      # 备份供应商示例（取消注释以启用轮换）
      # - provider: openai
      #   base_url: "https://api.openai.com/v1/chat/completions"
      #   auth_token: "sk-your-api-key"
      #   model: "gpt-4"

    # 启用 LLM 工具调用功能
    enabled_tool_call: false

    # 生成的最大令牌数
    max_tokens: 1000

    # 采样温度
    temperature: 0.7

    # Top-k 采样
    top_k: 50

    # Top-p 采样
    top_p: 0.9

    # 生成的补全数
    n: 1

    # 失败请求的最大重试次数
    max_retries: 3

    # API 请求超时时间
    timeout: 120s

    # 启用流式响应
    stream: false

    # 启用结构化 JSON 输出格式
    structured_json_format: false

    # LLM 的系统提示
    system_prompt: ""

    # API 请求的自定义标头
    custom_headers: ""
  ```


</details>

## 2. 使用设置 CLI

Osmedeus 通过 `osmedeus config` 命令提供 CLI 工具来管理配置文件。

![描述性替代文本](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/cli/cli-config-list.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=c785758b2c303328d6acd425b8a5e3b6)

<details>
<summary>完整使用示例</summary>

  ```bash
  base_folder = $HOME/osmedeus-base
  database.connection_timeout = 60
  database.db_engine = sqlite
  database.db_name = osmedeus
  database.db_path = {{base_folder}}/database-osm.sqlite
  database.host = localhost
  database.password = [REDACTED]
  database.port = 5432
  database.ssl_mode = disable
  database.username = osmedeus
  environments.external_agent_configs = {{base_folder}}/external-agent-configs
  environments.external_binaries_path = {{base_folder}}/external-binaries
  environments.external_configs = {{base_folder}}/external-configs
  environments.external_data = {{base_folder}}/external-data
  environments.markdown_report_templates = {{base_folder}}/markdown-report-templates
  environments.snapshot = {{base_folder}}/snapshot
  environments.workflows = {{base_folder}}/workflows
  environments.workspaces = $HOME/workspaces-osmedeus
  global_vars.CENSYS_API_KEY.as_env = [REDACTED]
  global_vars.CENSYS_API_KEY.value = [REDACTED]
  global_vars.GITHUB_API_KEY.as_env = [REDACTED]
  global_vars.GITHUB_API_KEY.value = [REDACTED]
  global_vars.PASSIVETOTAL_API_KEY.as_env = [REDACTED]
  global_vars.PASSIVETOTAL_API_KEY.value = [REDACTED]
  global_vars.SHODAN_API_KEY.as_env = [REDACTED]
  global_vars.SHODAN_API_KEY.value = [REDACTED]
  llm_config.custom_headers = 
  llm_config.enabled_tool_call = false
  llm_config.llm_providers.0.auth_token = [REDACTED]
  llm_config.llm_providers.0.base_url = http://localhost:11434/v1/chat/completions
  llm_config.llm_providers.0.model = gpt-oss:120b-cloud
  llm_config.llm_providers.0.provider = ollama
  llm_config.max_retries = 3
  llm_config.max_tokens = [REDACTED]
  llm_config.n = 1
  llm_config.stream = false
  llm_config.structured_json_format = false
  llm_config.system_prompt = 
  llm_config.temperature = 0.7
  llm_config.timeout = 120s
  llm_config.top_k = 50
  llm_config.top_p = 0.9
  notification.enabled = false
  notification.provider = telegram
  notification.telegram.bot_token = [REDACTED]
  notification.telegram.chat_id = 0
  notification.telegram.enabled = false
  redis.connection_timeout = 60
  redis.db = 0
  redis.host = 
  redis.password = [REDACTED]
  redis.port = 6379
  redis.username = 
  scan_tactic.aggressive = 40
  scan_tactic.default = 10
  scan_tactic.gently = 5
  server.host = 0.0.0.0
  server.jwt.expiration_minutes = 180
  server.jwt.secret_signing_key = [REDACTED]
  server.license = open-source
  server.password = [REDACTED]
  server.port = 8002
  server.simple_user_map_key.osmedeus = [REDACTED]
  server.ui_path = {{base_folder}}/ui/
  server.username = osmedeus
  server.workspace_prefix_key = [REDACTED]
  storage.access_key_id = [REDACTED]
  storage.bucket = 
  storage.enabled = false
  storage.endpoint = 
  storage.provider = s3
  storage.region = us-east-1
  storage.secret_access_key = [REDACTED]
  storage.use_ssl = true
  ```


</details>
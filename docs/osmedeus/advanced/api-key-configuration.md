> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 额外配置

> 安装 Osmedeus 的多种方式

本指南涵盖高级配置选项，包括二进制注册表、API 密钥、存储、LLM 和通知。

## 配置文件

默认位置：`~/osmedeus-base/osm-settings.yaml`

可通过以下方式覆盖：`osmedeus --settings-file /path/to/config.yaml`

***

## 向注册表添加新二进制文件

二进制注册表定义了可通过 `osmedeus install binary` 安装的工具。

### 注册表格式

创建或修改 `registry-metadata.json`：

```json theme={null}
{
  "mytool": {
    "desc": "mytool 的描述",
    "repo_link": "https://github.com/user/mytool",
    "version": "1.2.0",
    "tags": ["recon", "scanning"],
    "valide-command": "mytool --version",
    "linux": {
      "amd64": "https://github.com/user/mytool/releases/download/v1.2.0/mytool-linux-amd64.tar.gz",
      "arm64": "https://github.com/user/mytool/releases/download/v1.2.0/mytool-linux-arm64.tar.gz"
    },
    "darwin": {
      "amd64": "https://github.com/user/mytool/releases/download/v1.2.0/mytool-darwin-amd64.tar.gz",
      "arm64": "https://github.com/user/mytool/releases/download/v1.2.0/mytool-darwin-arm64.tar.gz"
    },
    "nix_package": "mytool"
  }
}
```

### 注册表字段

| 字段                             | 描述                       |
| --------------------------------- | --------------------------------- |
| `desc`                            | 工具描述                  |
| `repo_link`                       | 仓库 URL                    |
| `version`                         | 当前版本                   |
| `tags`                            | 用于过滤的分类          |
| `valide-command`                  | 验证安装的命令    |
| `linux`, `darwin`, `windows`      | 按操作系统/架构的下载 URL |
| `command-linux`, `command-darwin` | 替代安装命令      |
| `nix_package`                     | Nix 包名称                  |

### 使用自定义注册表

```bash theme={null}
# 使用本地注册表
osmedeus install binary -n mytool -r /path/to/registry.json

# 使用远程注册表
osmedeus install binary -n mytool -r https://example.com/registry.json
```

***

## API 密钥配置

在 `global_vars` 部分配置外部服务的 API 密钥。

### 配置

```yaml theme={null}
global_vars:
  # GitHub API（用于资产发现）
  GITHUB_API_KEY:
    value: "ghp_xxxxxxxxxxxx"
    as_env: true

  # Shodan API
  SHODAN_API_KEY:
    value: "xxxxxxxxxxxxxxxx"
    as_env: true

  # Censys API
  CENSYS_API_ID:
    value: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    as_env: true
  CENSYS_API_SECRET:
    value: "xxxxxxxxxxxxxxxxxxxx"
    as_env: true

  # SecurityTrails
  SECURITYTRAILS_API_KEY:
    value: "xxxxxxxxxxxxxxxxxxxx"
    as_env: true

  # VirusTotal
  VIRUSTOTAL_API_KEY:
    value: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    as_env: true

  # 自定义变量（仅工作流使用，不导出到环境变量）
  custom_wordlist:
    value: "/path/to/wordlist.txt"
    as_env: false
```

### 通过 CLI 设置

```bash theme={null}
osmedeus config set global_vars.GITHUB_API_KEY ghp_xxxx
osmedeus config set global_vars.SHODAN_API_KEY xxxx
```

### 在工作流中使用

```yaml theme={null}
# 作为模板变量
- command: 'curl -H "Authorization: token {{GITHUB_API_KEY}}" https://api.github.com/...'

# 作为环境变量（当 as_env: true 时）
- command: 'shodan search "hostname:{{Target}}"'  # 使用 $SHODAN_API_KEY
```

***

### 在工作流中使用

```yaml theme={null}
- name: upload-results
  type: function
  function: 'cdnUpload("{{Output}}/results.json", "scans/{{Target}}/results.json")'

- name: download-data
  type: function
  function: 'cdnDownload("wordlists/common.txt", "{{Output}}/wordlist.txt")'
```

***

## LLM 配置

为 `llm` 步骤类型配置 AI/LLM 供应商。

### 配置

```yaml theme={null}
llm_config:
  llm_providers:
    # 主供应商
    - provider: ollama
      base_url: "http://localhost:11434/v1/chat/completions"
      auth_token: ""
      model: "llama2"

    # 备用供应商
    - provider: openai
      base_url: "https://api.openai.com/v1/chat/completions"
      auth_token: "sk-xxxxxxxxxxxx"
      model: "gpt-4"

  # LLM 设置
  max_tokens: 1000
  temperature: 0.7
  top_p: 0.9
  max_retries: 3
  timeout: 120s
  stream: false
```

### 支持的供应商

| 供应商     | 基础 URL                                     |
| ------------ | -------------------------------------------- |
| Ollama       | `http://localhost:11434/v1/chat/completions` |
| OpenAI       | `https://api.openai.com/v1/chat/completions` |
| Azure OpenAI | `https://<resource>.openai.azure.com/...`    |
| 自定义       | 任何兼容 OpenAI 的端点               |

### 在工作流中使用

```yaml theme={null}
- name: analyze
  type: llm
  messages:
    - role: system
      content: "你是一名安全分析师。"
    - role: user
      content: "分析：{{Target}}"
  exports:
    analysis: "{{llm_step_content}}"
```

***

## 数据库配置

### SQLite（默认）

```yaml theme={null}
database:
  db_engine: sqlite
  db_path: "{{base_folder}}/database-osm.sqlite"
```

### PostgreSQL

```yaml theme={null}
database:
  db_engine: postgresql
  host: localhost
  port: 5432
  username: osmedeus
  password: secure_password
  db_name: osmedeus
  connection_timeout: 60
  ssl_mode: disable  # disable, require, verify-ca, verify-full
```

### 通过 CLI 设置

```bash theme={null}
osmedeus config set database.db_engine postgresql
osmedeus config set database.host localhost
osmedeus config set database.password mypassword
```

***

## 服务器身份验证

### 简单认证（用户名/密码）

```yaml theme={null}
server:
  simple_user_map_key:
    admin: "secure_password"
    readonly: "another_password"
```

### JWT 配置

```yaml theme={null}
server:
  jwt:
    secret_signing_key: "change-this-to-random-string"
    expiration_minutes: 180
```

### API 密钥认证

```yaml theme={null}
server:
  enabled_auth_api: true
  auth_api_key: "your-secure-api-key"
```

启用后，所有 API 请求都需要包含头部：`x-osm-api-key: your-secure-api-key`

### 通过 CLI 设置

```bash theme={null}
osmedeus config set server.username admin
osmedeus config set server.password secure123
osmedeus config set server.jwt.secret_signing_key random_key_here
osmedeus config set server.enabled_auth_api true
osmedeus config set server.auth_api_key my-api-key
```

***

## 扫描策略

配置不同扫描强度的线程数：

```yaml theme={null}
scan_tactic:
  aggressive: 40   # --tactic aggressive
  default: 10      # 默认行为
  gently: 5        # --tactic gently
```

用法：

```bash theme={null}
osmedeus run -m recon -t example.com --tactic aggressive
osmedeus run -m recon -t example.com --tactic gently
```

***

## 环境路径

```yaml theme={null}
environments:
  external_binaries_path: "{{base_folder}}/external-binaries"
  external_data: "{{base_folder}}/external-data"
  external_configs: "{{base_folder}}/external-configs"
  workspaces: "{{base_folder}}/workspaces"
  workflows: "{{base_folder}}/workflows"
  snapshot: "{{base_folder}}/snapshot"
```

***

## 快速设置清单

1. **安装二进制文件：** `osmedeus install binary --all`
2. **设置 API 密钥：** `osmedeus config set global_vars.GITHUB_API_KEY ghp_xxx`
3. **配置认证：** `osmedeus config set server.password secure123`
4. **测试配置：** `osmedeus config list`
5. **验证设置：** `osmedeus health`
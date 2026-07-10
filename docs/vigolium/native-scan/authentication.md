# Authenticated Scanning（身份验证扫描）

Vigolium 支持多会话身份验证扫描，通过两个可重复使用的标志 `--auth-file` 和 `--auth` 实现。这使得可以扫描登录墙后的内容，并检测授权绕过漏洞（IDOR/BOLA）。

身份验证文件可以使用 **YAML 或 JSON** 格式编写——格式通过文件扩展名（`.json`）或内容嗅探（以 `{` 或 `[` 开头）自动检测。

## Quick Start（快速开始）

```bash
# 内联会话（最简单）
vigolium scan https://app.com --auth "admin:Cookie:session_id=abc123"

# 单会话文件（YAML 或 JSON）
vigolium scan https://app.com --auth-file ./admin-session.yaml
vigolium scan https://app.com --auth-file ./admin-session.json

# 裸名称，解析自 scanning_strategy.session.session_dir
vigolium scan https://app.com --auth-file admin-session

# 包含多个会话的 Bundle 文件
vigolium scan https://app.com --auth-file ./auth-config.yaml
vigolium scan https://app.com --auth-file ./auth-config.json

# 混合使用——两个标志均可重复
vigolium scan https://app.com --auth-file admin --auth "compare:Cookie:sid=xyz"
```

## The Auth Flags（身份验证标志）

| 标志 | 接受类型 | 可重复 |
|------|---------|------------|
| `--auth-file <path>` | YAML/JSON 文件（单个会话 **或** `sessions:` bundle），或解析自 `scanning_strategy.session.session_dir` 的裸名称 | 是 |
| `--auth <name:Header:value>` | 内联会话——名称和标头作为静态标头注入 | 是 |

如果没有任何会话被显式标记为 `primary`，则第一个加载的会话将作为主会话。

## Session Roles（会话角色）

每个会话都有一个 **角色（role）**，决定其在扫描过程中的使用方式：

- **`primary`** — 主会话。用于内容发现、爬取，并在动态评估阶段作为默认请求器。应恰好有一个主会话。
- **`compare`** — 比较会话，用于 IDOR/BOLA 测试。在动态评估阶段，主会话发出的每个请求都会使用每个比较会话的凭据重新发送。如果比较会话能够访问本不应访问的资源，`authz-compare` 模块会将其标记为漏洞发现。

## Inline Sessions（内联会话）

`--auth` 标志接受 `name:Header:value` 格式的内联会话：

```bash
# 带 Cookie 的单个会话
vigolium scan https://app.com --auth "admin:Cookie:session_id=abc123"

# Bearer 令牌
vigolium scan https://app.com --auth "user1:Authorization:Bearer ***"

# 用于 IDOR 测试的多个会话
vigolium scan https://app.com \
  --auth "admin:Cookie:session=admin_token" \
  --auth "regular:Cookie:session=user_token"
```

`value` 部分中包含冒号的值也能正确处理——只有前两个冒号用作分隔符。

## Session Files（会话文件）

对于包含多个标头或登录流程的会话，请使用文件。支持 YAML 和 JSON 格式。文件可以在顶层包含单个会话 **或** `sessions:` 列表——加载器会自动检测结构。

### Static Headers（静态标头）

**YAML：**

```yaml
name: admin
role: primary
headers:
  Cookie: "session_id=abc123"
  Authorization: "Bearer mytoken"
```

**JSON：**

```json
{
  "name": "admin",
  "role": "primary",
  "headers": {
    "Cookie": "session_id=abc123",
    "Authorization": "Bearer mytoken"
  }
}
```

使用方法：

```bash
vigolium scan https://app.com --auth-file ./admin-session.yaml
vigolium scan https://app.com --auth-file ./admin-session.json
```

如果路径不是绝对路径，会话文件将从配置的 `session_dir`（默认 `~/.vigolium/sessions/`）解析。参见下方的 [Session Strategy Configuration](#session-strategy-configuration)。

### Login Flows（登录流程）

会话文件可以定义自动化的登录流程。扫描器在扫描开始时执行登录请求，并从响应中提取凭据。

**YAML：**

```yaml
name: admin
role: primary
login:
  url: "https://app.com/api/auth/login"
  method: POST
  content_type: "application/json"
  body: '{"username":"${ADMIN_USER}","password":"${ADMIN_PASS}"}'
  extract:
    - source: json
      path: "$.token"
      apply_as: "Authorization: Bearer ***"
```

**JSON：**

```json
{
  "name": "admin",
  "role": "primary",
  "login": {
    "url": "https://app.com/api/auth/login",
    "method": "POST",
    "content_type": "application/json",
    "body": "{\"username\":\"${ADMIN_USER}\",\"password\":\"${ADMIN_PASS}\"}",
    "extract": [
      {
        "source": "json",
        "path": "$.token",
        "apply_as": "Authorization: Bearer ***"
      }
    ]
  }
}
```

### Extraction Sources（提取源）

| 源 | 描述 | 示例 |
|--------|-------------|---------|
| `json` | 使用点符号从 JSON 响应体中提取值 | `path: "$.token"` |
| `cookie` | 从 `Set-Cookie` 响应标头中提取 Cookie。省略 `name` 则提取所有 Cookie | `name: "session_id"` |
| `header` | 从响应标头中提取值 | `name: "X-Auth-Token"` |

`apply_as` 字段定义如何将提取的值作为请求标头应用。使用 `{value}` 作为占位符。

## Bundle Files（Bundle 文件）

Bundle 文件在顶层的 `sessions:` 键下定义多个会话。通过 `--auth-file` 传入。

### YAML 格式

```yaml
sessions:
  # 主会话：JSON API 登录
  - name: admin
    role: primary
    login:
      url: "https://app.com/api/auth/login"
      method: POST
      content_type: "application/json"
      body: '{"username":"${ADMIN_USER}","password":"${ADMIN_PASS}"}'
      extract:
        - source: json
          path: "$.token"
          apply_as: "Authorization: Bearer ***"

  # 比较会话：基于表单的登录
  - name: regular_user
    role: compare
    login:
      url: "https://app.com/login"
      method: POST
      content_type: "application/x-www-form-urlencoded"
      body: "username=${USER_NAME}&password=${USER_PASS}"
      extract:
        - source: cookie

  # 比较会话：静态 API 密钥（无需登录）
  - name: api_key_user
    role: compare
    headers:
      X-API-Key: ***
```

### JSON 格式

```json
{
  "sessions": [
    {
      "name": "admin",
      "role": "primary",
      "login": {
        "url": "https://app.com/api/auth/login",
        "method": "POST",
        "content_type": "application/json",
        "body": "{\"username\":\"${ADMIN_USER}\",\"password\":\"${ADMIN_PASS}\"}",
        "extract": [
          {
            "source": "json",
            "path": "$.token",
            "apply_as": "Authorization: Bearer ***"
          }
        ]
      }
    },
    {
      "name": "regular_user",
      "role": "compare",
      "login": {
        "url": "https://app.com/login",
        "method": "POST",
        "content_type": "application/x-www-form-urlencoded",
        "body": "username=${USER_NAME}&password=${USER_PASS}",
        "extract": [
          {
            "source": "cookie"
          }
        ]
      }
    },
    {
      "name": "api_key_user",
      "role": "compare",
      "headers": {
        "X-API-Key": "${API_KEY}"
      }
    }
  ]
}
```

使用方法：

```bash
vigolium scan https://app.com --auth-file ./auth-config.yaml
vigolium scan https://app.com --auth-file ./auth-config.json
```

### When to Use JSON（何时使用 JSON）

在以下情况下，JSON 是不错的选择：

- **AI 代理生成会话配置**——大多数 LLM 生成的 JSON 比 YAML 更干净，代理模式（swarm、autopilot）本身已原生输出 JSON 格式的会话配置。
- **程序化生成**——脚本、CI 管道或构建会话配置的工具使用 JSON 通常更简单。
- **嵌入到其他 JSON 载荷中**——例如，REST API `POST /api/agent/run/autopilot` 的请求体包含作为嵌套 JSON 对象的会话配置。

对于手写配置，YAML 仍然更方便，因为注释和多行字符串有助于提高可读性。

### Format Detection（格式检测）

格式自动检测：

1. **文件扩展名**——`.json` 文件解析为 JSON；`.yaml` / `.yml` 解析为 YAML。
2. **内容嗅探**——如果扩展名不明确（或缺失），以 `{` 或 `[` 开头（去除空白后）的内容将解析为 JSON。
3. **回退**——其他所有内容解析为 YAML。

这意味着无扩展名的文件也能正常工作——直接传入 JSON 即可被检测：

```bash
# 从脚本生成配置，写入临时文件，进行扫描
./gen-auth-config.sh > /tmp/auth-config
vigolium scan https://app.com --auth-file /tmp/auth-config
```

## Session Config Schema Reference（会话配置模式参考）

YAML 和 JSON 使用相同的字段名。以下是完整模式：

```
SessionConfig
├── sessions[]              # 会话定义数组
│   ├── name                # (字符串，必需) 唯一会话名称
│   ├── role                # (字符串) "primary" 或 "compare"
│   ├── headers             # (映射) 静态标头，例如 {"Cookie": "sid=abc"}
│   ├── login               # (对象) 自动化登录流程
│   │   ├── url             # (字符串，必需) 登录端点 URL
│   │   ├── method          # (字符串，必需) HTTP 方法（POST、GET 等）
│   │   ├── content_type    # (字符串) 请求 Content-Type
│   │   ├── body            # (字符串) 请求体
│   │   └── extract[]       # (数组，必需) 凭据提取规则
│   │       ├── source      # (字符串) "json"、"cookie" 或 "header"
│   │       ├── name        # (字符串) 要提取的 Cookie/标头名称
│   │       ├── path        # (字符串) json 源的 JSONPath
│   │       └── apply_as    # (字符串) 标头模板，例如 "Authorization: Bearer ***"
│   └── login_request       # (字符串) 用于登录的原始 HTTP 请求（login 的替代方案）
```

每个会话只能设置 `headers`、`login` 或 `login_request` 中的一个。

## Environment Variables（环境变量）

会话文件（YAML 和 JSON）支持 `${VAR}` 语法用于机密信息。这可以将凭据保留在配置文件之外：

```bash
export ADMIN_USER=admin
export ADMIN_PASS=s3cret
vigolium scan https://app.com --auth-file ./auth-config.json
```

所有 `${VAR}` 引用在加载时从环境中展开，在格式解析之前进行。

## IDOR/BOLA Testing（IDOR/BOLA 测试）

要测试授权绕过漏洞，至少定义两个会话——一个主会话和一个或多个比较会话。

**YAML：**

```yaml
sessions:
  - name: admin
    role: primary
    headers:
      Cookie: "${ADMIN_SESSION_COOKIE}"

  - name: regular_user
    role: compare
    headers:
      Cookie: "${USER_SESSION_COOKIE}"

  # 可选：未认证会话
  - name: unauthenticated
    role: compare
```

**JSON：**

```json
{
  "sessions": [
    {
      "name": "admin",
      "role": "primary",
      "headers": {
        "Cookie": "${ADMIN_SESSION_COOKIE}"
      }
    },
    {
      "name": "regular_user",
      "role": "compare",
      "headers": {
        "Cookie": "${USER_SESSION_COOKIE}"
      }
    },
    {
      "name": "unauthenticated",
      "role": "compare"
    }
  ]
}
```

内置的 `authz-compare` 模块在存在比较会话时自动激活。它会使用比较会话的凭据重放主会话的请求，并标记指示访问控制失效的响应。

### How Detection Works（检测原理）

1. 主会话发出请求并收到响应（例如，`GET /api/users/42` -> 200 OK 并返回用户数据）。
2. 使用每个比较会话的凭据重放相同的请求。
3. 如果比较会话也收到成功响应且内容相似，该模块会报告一个潜在的 IDOR/BOLA 漏洞发现，严重级别为 **High**。

### Filtering to Auth Modules Only（仅过滤身份验证模块）

要仅运行授权测试而不运行其他活动模块：

```bash
vigolium scan https://app.com \
  --auth-file ./auth-config.json \
  --module-tag access-control
```

## How Sessions Affect Scan Phases（会话如何影响扫描阶段）

| 阶段 | 会话使用方式 |
|-------|---------------|
| 内容发现 / 爬取 | 仅主会话（由 `use_in_discovery` 控制） |
| 动态评估 | 主会话用于主要扫描；比较会话用于 IDOR/BOLA 重放（由 `compare_enabled` 控制） |

## Session Strategy Configuration（会话策略配置）

会话行为在 `vigolium-configs.yaml` 的 `scanning_strategy.session` 下配置（参见 `public/vigolium-configs.example.yaml` 获取完整的带注释示例）。

```yaml
scanning_strategy:
  session:
    # 会话文件存储目录。
    # 当 --auth-file 接收裸名称（例如 "myapp"）时，扫描器
    # 将其解析为 <session_dir>/myapp.yaml（或 .yml、.json）。
    # 默认：~/.vigolium/sessions/
    session_dir: ~/.vigolium/sessions/

    # 在内容发现和爬取阶段应用主会话标头。
    # 如果为 false，这些阶段以未认证方式运行，凭据仅
    # 在动态评估阶段使用。
    # 默认：true
    use_in_discovery: true

    # 启用使用比较会话的跨会话 IDOR/BOLA 重放。
    # 如果为 true 且定义了多个会话，authz-compare 模块
    # 会使用每个比较会话的凭据重放主会话请求。
    # 如果为 false，即使定义了比较会话也会被忽略。
    # 默认：true
    compare_enabled: true

    # 按此间隔重新执行登录流程以刷新即将过期的令牌。
    # 格式：Go 持续时间字符串（例如 "15m"、"1h"、"30m"）。
    # 默认：""（禁用——仅在扫描开始时登录一次）
    reauth_interval: ""

    # 当主会话收到以下 HTTP 状态码之一时，触发反应式
    # 重新认证。登录流程会立即重新执行，失败的请求会重试。
    # 默认：[]（禁用）
    reauth_on_status: []

    # 登录后 GET 的 URL，用于验证提取的凭据是否有效。
    # 扫描器在继续之前检查是否收到 2xx 响应。
    # 可以是相对路径（针对目标解析）或绝对 URL。
    # 默认：""（禁用）
    validate_url: ""
```

### Field Reference（字段参考）

| 字段 | 类型 | 默认值 | 描述 |
|-------|------|---------|-------------|
| `session_dir` | 字符串 | `~/.vigolium/sessions/` | 会话文件查找目录。`--auth-file myapp`（裸名称）解析为 `<session_dir>/myapp.yaml`（依次尝试 `.yaml`、`.yml`、`.json`）。支持 `~` 展开。 |
| `use_in_discovery` | 布尔值 | `true` | 当为 `true` 时，主会话的标头被注入到用于内容发现和爬取的请求器中。当为 `false` 时，这些阶段以未认证方式运行——适用于先映射公共攻击面，然后再扫描已认证内容。 |
| `compare_enabled` | 布尔值 | `true` | 当为 `true` 时，创建比较会话并激活 `authz-compare` 模块用于 IDOR/BOLA 测试。当为 `false` 时，即使定义了比较会话也会被忽略——当只需要已认证扫描而不需要授权比较时很方便。 |
| `reauth_interval` | 持续时间 | `""`（禁用） | Go 持续时间字符串（例如 `"15m"`、`"1h"`）。设置后，登录流程按此间隔重新执行，以刷新在扫描中期过期的令牌。 |
| `reauth_on_status` | []int | `[]`（禁用） | 触发反应式重新认证的 HTTP 状态码。当主会话收到这些状态码之一时，其登录流程会立即重新执行，并重试该请求。 |
| `validate_url` | 字符串 | `""`（禁用） | 登录后 GET 的相对或绝对 URL。扫描器检查是否收到 2xx 响应以确认凭据有效，然后再继续。可及早捕获错误的凭据。 |

### Session Directory Resolution（会话目录解析）

当 `--auth-file` 接收裸名称（无路径分隔符、无扩展名、少于 3 个冒号分隔的部分）时，扫描器从 `session_dir` 解析它。扩展名按顺序尝试：`.yaml`、`.yml`、`.json`。

```bash
# 当 session_dir 为 ~/.vigolium/sessions/ 时，以下命令等效
vigolium scan https://app.com --auth-file myapp
vigolium scan https://app.com --auth-file ~/.vigolium/sessions/myapp.yaml
vigolium scan https://app.com --auth-file ~/.vigolium/sessions/myapp.json
```

如果裸名称在任何扩展名下都没有匹配的文件，则默认附加 `.yaml`。绝对路径和包含目录分隔符的相对路径（例如 `./sessions/myapp.json`）绕过 `session_dir` 并按原样使用。

更改查找目录：

```yaml
scanning_strategy:
  session:
    session_dir: /opt/vigolium/shared-sessions/
```

### Common Patterns（常见模式）

**未认证发现，已认证扫描：**

```yaml
scanning_strategy:
  session:
    use_in_discovery: false
```

首先爬取面向公众的站点，然后仅在动态评估阶段应用会话标头。当您想先查看未认证攻击者能发现什么，然后再测试已认证攻击面时，这很有用。

**已认证扫描但不进行 IDOR 测试：**

```yaml
scanning_strategy:
  session:
    compare_enabled: false
```

当您只需要扫描登录墙后的内容，但没有多个用户角色进行比较时很有用。主会话的凭据应用于所有阶段，但不会创建比较请求器，`authz-compare` 模块保持非活动状态。

**带令牌刷新的长时间扫描：**

```yaml
scanning_strategy:
  session:
    reauth_interval: "30m"
    reauth_on_status: [401, 403]
    validate_url: "/api/whoami"
```

每 30 分钟主动重新执行登录流程，同时在收到 401 或 403 时也会被动触发。`validate_url` 在每次登录后确认凭据有效，然后再恢复扫描。

**团队共享会话目录：**

```yaml
scanning_strategy:
  session:
    session_dir: /shared/team/vigolium-sessions/
```

让所有团队成员指向共享目录，这样 `--auth-file staging-admin` 对所有人都解析到同一个文件。

扫描配置文件（`~/.vigolium/profiles/`）也可以覆盖会话策略值——对于拥有"快速未认证"配置文件和"深度已认证"配置文件的情况很有用。

## Using Session Config with Agent Modes（在 Agent 模式下使用会话配置）

Agent 模式（`swarm`、`autopilot`）可以通过源代码分析自动生成会话配置。生成的配置始终以 JSON 格式写入会话目录。

当使用 `--source` 运行 agent swarm 时，源代码分析阶段会发现代码库中的身份验证流程，并在会话目录中生成 `session-config.json`。然后该配置会自动馈送到后续的扫描阶段。

您也可以像常规扫描一样将预构建的会话配置传递给 agent 模式：

```bash
# 使用预配置的身份验证运行 Swarm
vigolium agent swarm \
  --target https://app.com \
  --auth-file ./auth-config.json
```

## Examples（示例）

### 使用 Bearer Token 扫描 REST API

```bash
vigolium scan https://api.example.com \
  --auth-file "admin:Authorization:Bearer ***"
```

### 使用基于 Cookie 的身份验证进行扫描

```bash
vigolium scan https://app.example.com \
  --auth-file "user:Cookie:PHPSESSID=abc123; csrftoken=xyz"
```

### 带登录自动化的完整 IDOR 测试（YAML）

```bash
export ADMIN_USER=admin ADMIN_PASS=admin123
export USER_NAME=user1 USER_PASS=user123

vigolium scan https://app.example.com \
  --auth-file ./auth-config.yaml \
  --module-tag access-control
```

### 带登录自动化的完整 IDOR 测试（JSON）

```bash
export ADMIN_USER=admin ADMIN_PASS=admin123
export USER_NAME=user1 USER_PASS=user123

vigolium scan https://app.example.com \
  --auth-file ./auth-config.json \
  --module-tag access-control
```

### 与其他扫描选项结合使用

身份验证标志可与所有其他扫描选项一起使用：

```bash
vigolium scan https://app.example.com \
  --auth-file ./auth-config.json \
  --strategy lite \
  --only dynamic-assessment \
  --concurrency 10 \
  --format html -o report.html
```

### 单行 JSON 身份验证配置

对于快速测试或 CI 脚本，您可以内联编写 JSON 配置：

```bash
echo '{"sessions":[{"name":"admin","role":"primary","headers":{"Authorization":"Bearer '\"$TOKEN\"'}}]}' > /tmp/auth.json
vigolium scan https://app.com --auth-file /tmp/auth.json
```

### Agent 生成的会话配置

当 AI 代理在源代码中发现身份验证流程时，它会生成如下 JSON：

```json
{
  "sessions": [
    {
      "name": "default_user",
      "role": "primary",
      "login": {
        "url": "https://app.com/api/login",
        "method": "POST",
        "content_type": "application/json",
        "body": "{\"email\":\"test@test.com\",\"password\":\"testpassword\"}",
        "extract": [
          {
            "source": "json",
            "path": "$.token",
            "apply_as": "Authorization: Bearer ***"
          }
        ]
      }
    }
  ]
}
```

这可以保存并在多次扫描中重复使用：

```bash
vigolium scan https://app.com --auth-file ./agent-generated-auth.json
```

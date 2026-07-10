---
title: "API 参考"
description: "错误码、分页、Cron 表达式、步骤类型"
---

# API 参考

## 错误响应

所有端点返回一致格式的错误：

```json
{
  "error": true,
  "message": "Error description"
}
```

**常见 HTTP 状态码：**
- `200` - 成功
- `201` - 已创建
- `202` - 已接受（异步操作已启动）
- `400` - 错误请求（输入无效）
- `401` - 未授权（令牌缺失或无效）
- `404` - 未找到
- `500` - 服务器内部错误

---

## 分页

返回列表的端点支持通过查询参数进行分页：

| 参数 | 默认值 | 最大值 | 描述 |
|-----------|---------|-----|-------------|
| `offset` | 0 | - | 要跳过的记录数 |
| `limit` | 20 | 10000 | 最大返回记录数 |

**示例：**
```bash
curl "http://localhost:8002/osm/api/assets?offset=100&limit=50" \
  -H "Authorization: Bearer ***"
```

---

## Cron 表达式参考

调度使用标准 cron 表达式：

```
┌───────────── 分钟 (0-59)
│ ┌───────────── 小时 (0-23)
│ │ ┌───────────── 日期 (1-31)
│ │ │ ┌───────────── 月份 (1-12)
│ │ │ │ ┌───────────── 星期 (0-6, 星期日=0)
│ │ │ │ │
* * * * *
```

**示例：**
- `0 2 * * *` - 每天凌晨 2:00
- `0 0 * * 0` - 每周日午夜
- `*/30 * * * *` - 每 30 分钟
- `0 9-17 * * 1-5` - 周一至周五，上午 9 点到下午 5 点每小时

---

## 工作流步骤类型

用于 YAML 工作流定义的工作流步骤类型的参考文档。

### bash

在本地系统或已配置的运行器上执行 Shell 命令。

```yaml
- name: run-nuclei
  type: bash
  log: "Running nuclei scan"
  command: nuclei -u {{Target}} -o {{Output}}/nuclei.txt
  timeout: 3600
  exports:
    nuclei_results: "{{Output}}/nuclei.txt"
```

**字段：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `name` | string | 是 | 唯一的步骤名称 |
| `type` | string | 是 | 必须为 `bash` |
| `command` | string | 否* | 要执行的单个命令 |
| `commands` | array | 否* | 顺序执行的命令 |
| `parallel_commands` | array | 否* | 并行运行的命令 |
| `timeout` | int | 否 | 超时时间（秒） |
| `log` | string | 否 | 执行期间显示的日志消息 |
| `pre_condition` | string | 否 | 必须为 true 才能运行的条件 |
| `exports` | map | 否 | 执行后要导出的变量 |

*`command`、`commands` 或 `parallel_commands` 中必须指定一个。

---

### function

通过 Otto VM 执行用 JavaScript 编写的工具函数。

```yaml
- name: check-file
  type: function
  log: "Checking if results exist"
  function: fileExists("{{Output}}/results.txt")
  exports:
    has_results: "{{check_file_output}}"
```

**字段：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `name` | string | 是 | 唯一的步骤名称 |
| `type` | string | 是 | 必须为 `function` |
| `function` | string | 否* | 要执行的单个函数 |
| `functions` | array | 否* | 顺序执行的函数 |
| `parallel_functions` | array | 否* | 并行运行的函数 |

*`function`、`functions` 或 `parallel_functions` 中必须指定一个。

---

### parallel-steps

并发运行多个步骤。

```yaml
- name: parallel-scans
  type: parallel-steps
  log: "Running scans in parallel"
  parallel_steps:
    - name: nuclei-scan
      type: bash
      command: nuclei -u {{Target}}
    - name: httpx-scan
      type: bash
      command: httpx -u {{Target}}
```

**字段：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `name` | string | 是 | 唯一的步骤名称 |
| `type` | string | 是 | 必须为 `parallel-steps` |
| `parallel_steps` | array | 是 | 要并发运行的步骤数组 |

---

### foreach

遍历文件或数组中的项目。

```yaml
- name: scan-subdomains
  type: foreach
  log: "Scanning each subdomain"
  input: "{{Output}}/subdomains.txt"
  variable: subdomain
  step:
    name: scan-subdomain
    type: bash
    command: httpx -u [[subdomain]]
```

**字段：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `name` | string | 是 | 唯一的步骤名称 |
| `type` | string | 是 | 必须为 `foreach` |
| `input` | string | 是 | 要遍历的文件路径或数组 |
| `variable` | string | 是 | 循环变量名（使用 `[[variable]]`） |
| `step` | object | 是 | 为每个项目执行的步骤 |
| `parallel` | int | 否 | 并行迭代数 |

---

### remote-bash

在 Docker 容器中或通过 SSH 执行命令。

```yaml
- name: docker-scan
  type: remote-bash
  log: "Running scan in Docker"
  step_runner: docker
  step_runner_config:
    image: "projectdiscovery/nuclei:latest"
    volumes:
      - "{{Output}}:/output"
  command: nuclei -u {{Target}} -o /output/nuclei.txt
```

**字段：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `name` | string | 是 | 唯一的步骤名称 |
| `type` | string | 是 | 必须为 `remote-bash` |
| `step_runner` | string | 是 | `docker` 或 `ssh` |
| `step_runner_config` | object | 否 | 运行器特定配置 |
| `command` | string | 否* | 要执行的命令 |
| `commands` | array | 否* | 顺序执行的命令 |

**Docker 配置：**
```yaml
step_runner_config:
  image: "image:tag"
  volumes: ["host:container"]
  env:
    KEY: value
  network: "host"
  workdir: "/app"
```

**SSH 配置：**
```yaml
step_runner_config:
  host: "worker.example.com"
  user: "ubuntu"
  key_file: "~/.ssh/id_rsa"
  port: 22
```

---

### http

发起 HTTP 请求并捕获响应。

```yaml
- name: api-request
  type: http
  log: "Calling API"
  url: "https://api.example.com/endpoint"
  method: POST
  headers:
    Content-Type: "application/json"
    Authorization: "Bearer {{api_token}}"
  request_body: '{"domain": "{{Target}}"}'
  timeout: 30
  exports:
    api_response: "{{api_request_body}}"
```

**字段：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `name` | string | 是 | 唯一的步骤名称 |
| `type` | string | 是 | 必须为 `http` |
| `url` | string | 是 | 请求 URL |
| `method` | string | 否 | HTTP 方法（默认：GET） |
| `headers` | map | 否 | 请求头 |
| `request_body` | string | 否 | POST/PUT 的请求体 |
| `timeout` | int | 否 | 超时时间（秒） |

**自动导出：**
- `<step_name>_status_code` - HTTP 状态码
- `<step_name>_body` - 响应体
- `<step_name>_headers` - 响应头

---

### llm

执行 LLM（大语言模型）API 调用以进行 AI 驱动的分析。

```yaml
- name: analyze-target
  type: llm
  log: "Analyzing target with LLM"
  messages:
    - role: system
      content: "You are a security analyst."
    - role: user
      content: "Analyze the security of {{Target}}"
  llm_config:
    max_tokens: 1000
    temperature: 0.7
  timeout: 60
  exports:
    analysis: "{{analyze_target_content}}"
```

**字段：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `name` | string | 是 | 唯一的步骤名称 |
| `type` | string | 是 | 必须为 `llm` |
| `messages` | array | 否* | 聊天消息（role + content） |
| `is_embedding` | bool | 否 | 设置为 true 以生成嵌入 |
| `embedding_input` | array | 否* | 要嵌入的字符串 |
| `llm_config` | object | 否 | 步骤级别的 LLM 配置 |
| `tools` | array | 否 | 用于函数调用的工具定义 |
| `tool_choice` | string | 否 | 工具选择（`auto`、`none` 等） |
| `extra_llm_parameters` | map | 否 | 额外的供应商特定参数 |
| `timeout` | int | 否 | 超时时间（秒） |

*`messages` 或 `embedding_input`（配合 `is_embedding: true`）中必须指定一个。

**消息格式：**
```yaml
messages:
  - role: system
    content: "System prompt"
  - role: user
    content: "User message with {{variables}}"
```

**多模态消息（带图片）：**
```yaml
messages:
  - role: user
    content:
      - type: text
        text: "What do you see in this screenshot?"
      - type: image_url
        image_url:
          url: "data:image/png;base64,{{screenshot_base64}}"
```

**LLM 配置覆盖：**
```yaml
llm_config:
  model: "gpt-4"
  max_tokens: 2000
  temperature: 0.3
  response_format:
    type: json_object
```

**嵌入：**
```yaml
- name: generate-embeddings
  type: llm
  is_embedding: true
  embedding_input:
    - "{{Target}} security analysis"
    - "vulnerability assessment"
  exports:
    embeddings: "{{generate_embeddings_llm_resp}}"
```

**工具调用：**
```yaml
- name: with-tools
  type: llm
  messages:
    - role: user
      content: "What DNS records exist for {{Target}}?"
  tools:
    - type: function
      function:
        name: dns_lookup
        description: "Look up DNS records"
        parameters:
          type: object
          properties:
            domain:
              type: string
          required: [domain]
  tool_choice: auto
```

**自动导出：**
- `<step_name>_llm_resp` - 完整响应对象（id、model、usage、content、tool_calls）
- `<step_name>_content` - 仅内容字符串，便于访问

> ## Documentation Index
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 步骤类型

> 工作流执行可用的步骤类型

Osmedeus 支持 8 种步骤类型，以满足不同的执行需求。

## 概述

| 类型               | 描述                     | 主要用途                         |
| ------------------ | ------------------------ | -------------------------------- |
| `bash`             | 执行 shell 命令          | 运行工具、文件操作               |
| `function`         | 运行工具函数             | 条件判断、日志记录、文件检查     |
| `foreach`          | 遍历文件行               | 处理列表                         |
| `parallel-steps`   | 并发运行步骤             | 并行工具执行                     |
| `remote-bash`      | 按步骤执行 Docker/SSH    | 混合环境                         |
| `http`             | 发起 HTTP 请求           | API 调用、Webhook                |
| `llm`              | AI 驱动处理              | 分析、总结                       |
| `agent`            | 自主 LLM 执行            | 自主工具调用代理                 |

## bash

执行 shell 命令。

### 基本命令

```yaml theme={null}
- name: run-subfinder
  type: bash
  command: subfinder -d {{target}} -o {{Output}}/subs.txt
```

### 多个命令（顺序执行）

```yaml theme={null}
- name: setup
  type: bash
  commands:
    - mkdir -p {{Output}}/scans
    - echo "Starting scan for {{target}}"
    - date > {{Output}}/start-time.txt
```

### 并行命令

```yaml theme={null}
- name: run-tools
  type: bash
  parallel_commands:
    - subfinder -d {{target}} -o {{Output}}/subfinder.txt
    - amass enum -passive -d {{target}} -o {{Output}}/amass.txt
    - assetfinder {{target}} > {{Output}}/assetfinder.txt
```

### 结构化参数

```yaml theme={null}
- name: nuclei-scan
  type: bash
  command: nuclei
  input_args:
    - name: target-list
      flag: -l
      value: "{{Output}}/live.txt"
  output_args:
    - name: output
      flag: -o
      value: "{{Output}}/nuclei.txt"
  config_args:
    - name: templates
      flag: -t
      value: "{{Data}}/templates/cves"
  speed_args:
    - name: rate-limit
      flag: -rl
      value: "150"
```

### 将输出保存到文件

```yaml theme={null}
- name: scan
  type: bash
  command: nmap -sV {{target}}
  std_file: "{{Output}}/nmap-output.txt"
```

## function

通过 Goja JavaScript VM 执行工具函数。

### 单个函数

```yaml theme={null}
- name: log-start
  type: function
  function: log_info("Starting scan for {{target}}")
```

### 多个函数

```yaml theme={null}
- name: check-files
  type: function
  functions:
    - log_info("Checking prerequisites")
    - fileExists("{{Output}}/targets.txt")
    - log_info("Ready to proceed")
```

### 并行函数

```yaml theme={null}
- name: parallel-checks
  type: function
  parallel_functions:
    - fileLength("{{Output}}/subs.txt")
    - fileLength("{{Output}}/urls.txt")
    - fileLength("{{Output}}/live.txt")
```

### 在条件中使用

```yaml theme={null}
- name: run-if-exists
  type: bash
  pre_condition: 'fileExists("{{Output}}/targets.txt")'
  command: nuclei -l {{Output}}/targets.txt
```

## foreach

使用工作池并行遍历文件中的行。

### 基本循环

```yaml theme={null}
- name: probe-subdomains
  type: foreach
  input: "{{Output}}/subdomains.txt"
  variable: subdomain
  threads: 10
  step:
    name: httpx-probe
    type: bash
    command: echo [[subdomain]] | httpx -silent >> {{Output}}/live.txt
```

### 嵌套变量

```yaml theme={null}
- name: scan-hosts
  type: foreach
  input: "{{Output}}/hosts.txt"
  variable: host
  threads: 5
  step:
    name: nuclei-scan
    type: bash
    command: nuclei -u [[host]] -t {{templates}} -o {{Output}}/nuclei-[[host]].txt
```

### 有界并发

foreach 执行器使用工作池模式：

```
┌───────────────────────────────────────────────────┐
│                  Foreach 执行器                     │
│                                                    │
│   输入文件: subdomains.txt (1000 行)               │
│                      │                             │
│                      ▼                             │
│   ┌────────────────────────────────────────────┐  │
│   │            工作池 (threads: 10)             │  │
│   │  ┌────┐ ┌────┐ ┌────┐ ... ┌────┐          │  │
│   │  │ W1 │ │ W2 │ │ W3 │     │ W10│          │  │
│   │  └────┘ └────┘ └────┘     └────┘          │  │
│   └────────────────────────────────────────────┘  │
│                      │                             │
│                      ▼                             │
│   所有项目处理完成后收集结果                         │
│                                                    │
└───────────────────────────────────────────────────┘
```

* 工作线程从共享队列中拉取项目
* 最多同时处理 `threads` 个项目
* 内存高效：不会预先启动所有 goroutine
* 上下文超时时优雅取消

### 字段

| 字段                    | 必需 | 描述                                                         |
| ----------------------- | ---- | ------------------------------------------------------------ |
| `input`                 | 是   | 包含项目的文件路径（每行一个）                               |
| `variable`              | 是   | 当前项目的变量名                                             |
| `threads`               | 否   | 最大并发迭代数（默认：1）                                    |
| `step`                  | 是   | 为每个项目执行的步骤                                         |
| `variable_pre_process`  | 否   | 在存储前转换每个输入行（例如 `trim([[line]])`）              |

### 输入预处理

在将每个输入行存储到循环变量之前进行转换：

```yaml theme={null}
- name: scan-cleaned-hosts
  type: foreach
  input: "{{Output}}/hosts.txt"
  variable: host
  variable_pre_process: "trim([[line]])"
  threads: 10
  step:
    name: probe
    type: bash
    command: httpx -u [[host]] -silent
```

### 变量语法

使用 `[[variable]]`（双括号）表示循环变量，以避免与 `{{templates}}` 冲突：

```yaml theme={null}
step:
  name: scan
  type: bash
  # [[subdomain]] - 每次迭代替换
  # {{Output}} - 循环前解析一次
  command: nuclei -u [[subdomain]] -o {{Output}}/result-[[subdomain]].txt
```

### 嵌套 Foreach

Foreach 步骤可以包含其他 foreach 步骤：

```yaml theme={null}
- name: scan-ports-per-host
  type: foreach
  input: "{{Output}}/hosts.txt"
  variable: host
  threads: 5
  step:
    name: scan-ports
    type: foreach
    input: "{{Output}}/ports.txt"
    variable: port
    threads: 10
    step:
      name: probe
      type: bash
      command: nc -zv [[host]] [[port]]
```

## parallel-steps

并发运行多个步骤。

```yaml theme={null}
- name: parallel-recon
  type: parallel-steps
  parallel_steps:
    - name: subfinder
      type: bash
      command: subfinder -d {{target}} -o {{Output}}/subfinder.txt

    - name: amass
      type: bash
      command: amass enum -passive -d {{target}} -o {{Output}}/amass.txt

    - name: findomain
      type: bash
      command: findomain -t {{target}} -o {{Output}}/findomain.txt
```

嵌套步骤可以是任何类型：

```yaml theme={null}
- name: parallel-checks
  type: parallel-steps
  parallel_steps:
    - name: check-dns
      type: bash
      command: dig {{target}}

    - name: log-check
      type: function
      function: log_info("Parallel check running")

    - name: probe-hosts
      type: foreach
      input: "{{Output}}/subs.txt"
      variable: sub
      threads: 5
      step:
        type: bash
        command: echo [[sub]] | httpx
```

## remote-bash

在 Docker 或 SSH 中执行命令，无需模块级 Runner。

### Docker 执行

```yaml theme={null}
- name: docker-nuclei
  type: remote-bash
  step_runner: docker
  step_runner_config:
    image: projectdiscovery/nuclei:latest
    volumes:
      - "{{Output}}:/output"
    environment:
      - "API_KEY={{api_key}}"
  command: nuclei -u {{target}} -o /output/nuclei.txt
```

### SSH 执行

```yaml theme={null}
- name: ssh-nmap
  type: remote-bash
  step_runner: ssh
  step_runner_config:
    host: "{{ssh_host}}"
    port: 22
    user: "{{ssh_user}}"
    key_file: ~/.ssh/scanner_key
  command: nmap -sV {{target}} -oN /tmp/nmap.txt
  step_remote_file: /tmp/nmap.txt
  host_output_file: "{{Output}}/nmap-result.txt"
```

### 字段

| 字段                  | 必需 | 描述                           |
| --------------------- | ---- | ------------------------------ |
| `step_runner`         | 是   | `docker` 或 `ssh`              |
| `step_runner_config`  | 是   | Runner 配置                    |
| `command`             | 是   | 要执行的命令                   |
| `step_remote_file`    | 否   | 要复制回来的远程文件           |
| `host_output_file`    | 否   | 远程文件的本地目标路径         |

## http

发起 HTTP 请求，支持自动重试和连接池。

### 支持的方法

| 方法     | 描述                 |
| -------- | -------------------- |
| `GET`    | 检索数据（默认）     |
| `POST`   | 创建/提交数据        |
| `PUT`    | 更新/替换资源        |
| `PATCH`  | 部分更新             |
| `DELETE` | 删除资源             |

### GET 请求

```yaml theme={null}
- name: fetch-api
  type: http
  url: "https://api.example.com/data/{{target}}"
  method: GET
  headers:
    Authorization: "Bearer {{api_token}}"
  exports:
    api_response: "{{fetch_api_http_resp.response_body}}"
    status: "{{fetch_api_http_resp.status_code}}"
```

### POST 请求

```yaml theme={null}
- name: submit-scan
  type: http
  url: "https://scanner.example.com/api/scan"
  method: POST
  headers:
    Content-Type: application/json
  request_body: |
    {
      "target": "{{target}}",
      "scan_type": "full"
    }
```

### PUT 请求

```yaml theme={null}
- name: update-config
  type: http
  url: "https://api.example.com/config/{{target}}"
  method: PUT
  headers:
    Content-Type: application/json
    Authorization: "Bearer {{api_token}}"
  request_body: '{"enabled": true}'
```

### PATCH 请求

```yaml theme={null}
- name: patch-status
  type: http
  url: "https://api.example.com/scan/{{scan_id}}"
  method: PATCH
  headers:
    Content-Type: application/json
  request_body: '{"status": "completed"}'
```

### DELETE 请求

```yaml theme={null}
- name: remove-entry
  type: http
  url: "https://api.example.com/entries/{{entry_id}}"
  method: DELETE
  headers:
    Authorization: "Bearer {{api_token}}"
```

### 自动导出的变量

HTTP 步骤执行后，变量以 `<step_name>_http_resp` 模式导出：

```yaml theme={null}
# 访问方式：{{step_name_http_resp.field}}
# 可用字段：
#   status_code      - HTTP 状态码（整数）
#   response_body    - 响应体（字符串）
#   response_headers - 响应头（映射）
#   content_length   - 响应大小（字节，整数）
#   response_time_ms - 请求持续时间（毫秒，整数）
#   error            - 失败时的错误消息（字符串或 null）
#   message          - 状态消息（字符串）
```

### HTTP 特性

* **连接池**：复用连接以提高效率
* **自动重试**：在网络错误和 5xx 响应时重试（最多 3 次）
* **超时**：可通过步骤的 `timeout` 字段配置（默认：30s）
* **模板支持**：请求头和请求体支持 `{{variable}}` 插值

## llm

使用 LLM API（兼容 OpenAI）进行 AI 驱动处理。

### 聊天补全

```yaml theme={null}
- name: analyze-findings
  type: llm
  messages:
    - role: system
      content: You are a security analyst. Analyze the findings and provide a summary.
    - role: user
      content: |
        Analyze these vulnerability findings:
        {{readFile("{{Output}}/vulnerabilities.txt")}}
  exports:
    analysis: "{{analyze_findings_content}}"
```

### 消息角色

| 角色        | 描述                     |
| ----------- | ------------------------ |
| `system`    | 定义行为的系统提示       |
| `user`      | 用户输入/问题            |
| `assistant` | 模型之前的响应           |
| `tool`      | 工具/函数调用结果        |

### 工具调用

定义 LLM 可以调用的工具（兼容 OpenAI 的函数调用）：

```yaml theme={null}
- name: intelligent-scan
  type: llm
  messages:
    - role: system
      content: You are a security scanner assistant.
    - role: user
      content: Analyze {{target}} and suggest next steps.
  tools:
    - type: function
      function:
        name: run_scan
        description: Execute a security scan
        parameters:
          type: object
          properties:
            scan_type:
              type: string
              enum: [port, vuln, web]
            target:
              type: string
              description: Target to scan
          required:
            - scan_type
            - target
  tool_choice: auto  # auto, none, 或 {"type": "function", "function": {"name": "run_scan"}}
```

工具调用可通过 `{{step_name_llm_resp.tool_calls}}` 在导出中使用。

### 嵌入

为文本生成向量嵌入：

```yaml theme={null}
- name: generate-embeddings
  type: llm
  is_embedding: true
  embedding_input:
    - "{{readFile('{{Output}}/finding1.txt')}}"
    - "{{readFile('{{Output}}/finding2.txt')}}"
  exports:
    embeddings: "{{generate_embeddings_llm_resp.embeddings}}"
```

### 多模态内容（视觉）

在消息中包含图像：

```yaml theme={null}
- name: analyze-screenshot
  type: llm
  messages:
    - role: user
      content:
        - type: text
          text: "Analyze this application screenshot for security issues"
        - type: image_url
          image_url:
            url: "{{Output}}/screenshot.png"
```

### 结构化输出（JSON Schema）

强制结构化 JSON 响应：

```yaml theme={null}
- name: structured-analysis
  type: llm
  messages:
    - role: system
      content: You are a security analyst.
    - role: user
      content: Analyze {{target}} and return structured findings.
  llm_config:
    response_format:
      type: json_schema
      json_schema:
        name: security_findings
        schema:
          type: object
          properties:
            severity:
              type: string
              enum: [critical, high, medium, low, info]
            findings:
              type: array
              items:
                type: object
                properties:
                  title:
                    type: string
                  description:
                    type: string
          required:
            - severity
            - findings
```

### 配置覆盖

按步骤覆盖全局 LLM 设置：

```yaml theme={null}
- name: custom-llm
  type: llm
  llm_config:
    model: gpt-4-turbo
    max_tokens: 4000
    temperature: 0.3
    top_p: 0.95
    timeout: 5m
    max_retries: 5
    custom_headers:
      X-Custom-Header: value
  messages:
    - role: user
      content: Analyze {{target}}
```

### 自动导出的变量

LLM 步骤执行后：

```yaml theme={null}
# 访问方式：{{step_name_llm_resp.field}} 或 {{step_name_content}}
# step_name_llm_resp 中的可用字段：
#   id             - 响应 ID
#   model          - 使用的模型
#   content        - 响应文本（第一个选择）
#   finish_reason  - 生成停止的原因
#   role           - 消息角色
#   tool_calls     - 工具调用（如果有，数组）
#   all_contents   - 所有选择（如果 n > 1）
#   usage          - Token 使用量（prompt_tokens, completion_tokens, total_tokens）
#   embeddings     - 嵌入向量（嵌入模式）
#
# 简写：{{step_name_content}} 直接包含响应文本
```

### 供应商轮换

如果配置了多个 LLM 供应商，执行器会自动：

* 在遇到速率限制或错误时轮换到下一个供应商
* 最多重试 `max_retries * provider_count` 次
* 记录速率限制指标以供监控

## agent

自主 LLM 执行，带有自主工具调用循环。代理接收任务，规划方法，迭代调用工具，并生成最终答案。

### 基本用法

```yaml theme={null}
- name: analyze-target
  type: agent
  query: "Enumerate subdomains of {{Target}} and summarize findings."
  system_prompt: "You are a security reconnaissance agent."
  max_iterations: 10
  agent_tools:
    - preset: bash
    - preset: read_file
    - preset: save_content
  exports:
    findings: "{{agent_content}}"
```

### 预设工具

以下预设工具可通过 `preset` 字段使用：

| 预设               | 描述                           |
| ------------------ | ------------------------------ |
| `bash`             | 执行 shell 命令                |
| `read_file`        | 读取整个文件内容               |
| `read_lines`       | 读取文件中的指定行范围         |
| `file_exists`      | 检查文件是否存在               |
| `file_length`      | 统计文件中的非空行数           |
| `append_file`      | 向文件追加内容                 |
| `save_content`     | 将内容写入文件                 |
| `glob`             | 查找匹配 glob 模式的文件       |
| `grep_string`      | 在文件中搜索字面字符串         |
| `grep_regex`       | 在文件中搜索正则模式           |
| `http_get`         | 发起 HTTP GET 请求             |
| `http_request`     | 发起 HTTP 请求（任意方法）     |
| `jq`               | 使用 jq 表达式查询 JSON        |
| `exec_python`      | 执行内联 Python 代码           |
| `exec_python_file` | 执行 Python 文件               |
| `exec_ts`          | 执行内联 TypeScript 代码       |
| `exec_ts_file`     | 执行 TypeScript 文件           |
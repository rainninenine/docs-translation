> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# LLM 与 Agent 集成

使用大语言模型的 AI 驱动工作流步骤。

## 概述

Osmedeus 提供三种用于 LLM 集成的步骤类型：

* **`llm`** — 单次 LLM 调用：聊天补全、工具调用、嵌入、多模态内容、结构化输出
* **`agent`** — 代理执行循环：迭代工具调用、子代理、内存管理、规划阶段、多目标执行
* **`agent-acp`** — 通过 [Agent 通信协议 (ACP)](https://github.com/anthropics/agent-communication-protocol) 执行外部 AI 代理：委托给真实的代理二进制文件（Claude Code、Codex、OpenCode、Gemini）

## 设置

### 配置

在 `osm-settings.yaml` 中，在 `llm_providers` 下配置一个或多个供应商。供应商会在请求间自动轮换：

```yaml
llm:
  llm_providers:
    - provider: openai
      base_url: "https://api.openai.com/v1"
      auth_token: "sk-..."
      model: gpt-4
    - provider: anthropic
      base_url: "https://api.anthropic.com/v1"
      auth_token: "sk-ant-..."
      model: claude-3-opus
  max_tokens: 4096
  temperature: 0.7
  stream: false
```

### 环境变量

环境变量会覆盖默认供应商的设置：

```bash
export OSM_LLM_BASE_URL=https://api.openai.com/v1
export OSM_LLM_AUTH_TOKEN=sk-...
export OSM_LLM_MODEL=gpt-4
```

## 聊天补全

### 基本用法

```yaml
- name: analyze-results
  type: llm
  messages:
    - role: system
      content: 你是一名安全分析师。请简洁地分析发现结果。
    - role: user
      content: |
        分析以下漏洞：
        {{readFile("{{Output}}/vulns.txt")}}
  exports:
    analysis: "{{analyze_results_content}}"
```

!!! note "  导出变量基于**清理后的步骤名称**（连字符替换为下划线）。
  名为 `analyze-results` 的步骤会生成导出 `analyze_results_llm_resp`（完整响应对象）和 `analyze_results_content`（仅文本内容）。"

### 消息角色

| 角色        | 描述               |
| ----------- | ------------------ |
| `system`    | 系统指令           |
| `user`      | 用户输入           |
| `assistant` | 之前的 AI 响应     |
| `tool`      | 工具调用结果       |

### 多轮对话

```yaml
- name: chat
  type: llm
  messages:
    - role: system
      content: 你是一名有用的安全助手。
    - role: user
      content: 什么是 SQL 注入？
    - role: assistant
      content: SQL 注入是一种代码注入技术...
    - role: user
      content: 如何在 Python 中预防它？
```

## 工具调用

### 定义工具

```yaml
- name: intelligent-scan
  type: llm
  messages:
    - role: system
      content: 你是一名安全扫描器。使用工具分析目标。
    - role: user
      content: 分析 {{target}} 的安全问题。
  tools:
    - type: function
      function:
        name: port_scan
        description: 扫描目标上的端口
        parameters:
          type: object
          properties:
            target:
              type: string
              description: 目标 IP 或主机名
            ports:
              type: string
              description: 端口范围（例如 "1-1000"）
          required: ["target"]

    - type: function
      function:
        name: vulnerability_scan
        description: 运行漏洞扫描
        parameters:
          type: object
          properties:
            target:
              type: string
            templates:
              type: string
              enum: ["cves", "misconfigurations", "exposures"]
          required: ["target"]
```

### 处理工具调用

工具调用在 `_llm_resp` 对象中导出：

```yaml
- name: ai-scan
  type: llm
  messages:
    - role: user
      content: 扫描 {{target}}
  tools:
    - type: function
      function:
        name: scan
        parameters: { ... }
  exports:
    full_response: "{{ai_scan_llm_resp}}"

- name: execute-tool
  type: function
  pre_condition: '{{full_response}} != ""'
  function: |
    // 解析并执行工具调用
    executeToolCalls("{{full_response}}")
```

## 嵌入

### 生成嵌入

```yaml
- name: embed-findings
  type: llm
  is_embedding: true
  embedding_input:
    - "登录表单中的 SQL 注入"
    - "搜索中的跨站脚本"
    - "不安全的直接对象引用"
  exports:
    embeddings: "{{embed_findings_llm_resp}}"
```

### 与文件一起使用

```yaml
- name: embed-vulns
  type: llm
  is_embedding: true
  embedding_input: "{{readLines('{{Output}}/vulns.txt')}}"
  exports:
    vuln_embeddings: "{{embed_vulns_llm_resp}}"
```

## 结构化输出

### JSON Schema

```yaml
- name: extract-findings
  type: llm
  messages:
    - role: user
      content: |
        从以下报告中提取漏洞：
        {{readFile("{{Output}}/scan-report.txt")}}
  response_format:
    type: json_schema
    json_schema:
      name: vulnerabilities
      schema:
        type: object
        properties:
          findings:
            type: array
            items:
              type: object
              properties:
                title:
                  type: string
                severity:
                  type: string
                  enum: ["critical", "high", "medium", "low"]
                description:
                  type: string
        required: ["findings"]
  exports:
    structured_findings: "{{extract_findings_content}}"
```

## 配置覆盖

### 每步配置

```yaml
- name: local-analysis
  type: llm
  llm_config:
    provider: ollama
    model: llama2
    max_tokens: 2048
    temperature: 0.5
    stream: true
  messages:
    - role: user
      content: 分析 {{target}}
```

### 额外参数

```yaml
- name: creative-analysis
  type: llm
  messages:
    - role: user
      content: 为 {{target}} 编写安全评估
  extra_llm_parameters:
    temperature: 0.9
    top_p: 0.95
    frequency_penalty: 0.5
```

## 多模态内容

### 图像分析

```yaml
- name: analyze-screenshot
  type: llm
  messages:
    - role: user
      content:
        - type: text
          text: 分析此截图的安全问题。
        - type: image_url
          image_url:
            url: "file://{{Output}}/screenshot.png"
```

## 流式输出

`llm` 和 `agent` 步骤都支持通过 `stream` 字段进行流式输出：

```yaml
- name: stream-analysis
  type: llm
  stream: true
  messages:
    - role: user
      content: 详细分析 {{target}}。
```

`stream` 字段会覆盖 `llm_config.stream` 和全局配置设置。

***

## Agent 步骤类型

`agent` 步骤类型提供了一个**代理 LLM 执行循环**——LLM 迭代调用工具、处理结果并进行推理，直到完成。这与单次 `llm` 步骤有本质区别。

### 基本 Agent

```yaml
- name: recon-agent
  type: agent
  query: "枚举 {{Target}} 的子域名并识别感兴趣的服务。"
  system_prompt: "你是一名专业的安全侦察代理。"
  max_iterations: 15
  agent_tools:
    - preset: bash
    - preset: read_file
    - preset: save_content
  exports:
    findings: "{{agent_content}}"
```

| 字段              | 类型            | 必需    | 描述                                        |
| ---------------- | --------------- | ------- | ------------------------------------------- |
| `query`          | string          | 是\*    | 代理的任务提示                              |
| `queries`        | string\[]       | 是\*    | 按顺序执行的多个目标                        |
| `system_prompt`  | string          | 否      | 代理的系统提示                              |
| `max_iterations` | int             | 是      | 最大工具调用循环迭代次数（必须 > 0）        |
| `agent_tools`    | AgentToolDef\[] | 否      | 代理可用的工具                              |

!!! note "需要 `query`（单目标）或 `queries`（多目标）之一，不能同时使用。"

### 预设工具

预设工具引用 Osmedeus 内置函数，并自动生成 schema：

```yaml
agent_tools:
  - preset: bash
  - preset: read_file
  - preset: grep_regex
```

| 预设               | 描述                                                   |
| ------------------ | ------------------------------------------------------ |
| `bash`             | 执行 shell 命令并返回输出                              |
| `read_file`        | 读取文件内容                                           |
| `read_lines`       | 读取文件并以行数组形式返回内容                         |
| `file_exists`      | 检查给定路径的文件是否存在                             |
| `file_length`      | 计算文件中非空行的数量                                 |
| `append_file`      | 将源文件内容追加到目标文件                             |
| `save_content`     | 将字符串内容写入文件（如果存在则覆盖）                 |
| `glob`             | 查找匹配 glob 模式的文件                               |
| `grep_string`      | 在文件中搜索包含指定字符串的行                         |
| `grep_regex`       | 在文件中搜索匹配正则表达式的行                         |
| `http_get`         | 发起 HTTP GET 请求并返回响应                           |
| `http_request`     | 使用指定方法、标头和主体发起 HTTP 请求                 |
| `jq`               | 使用 jq 表达式语法查询 JSON 数据                       |
| `exec_python`      | 运行内联 Python 代码并返回 stdout                      |
| `exec_python_file` | 运行 Python 文件并返回 stdout                          |
| `exec_ts`          | 通过 bun 运行内联 TypeScript 代码并返回 stdout         |
| `exec_ts_file`     | 通过 bun 运行 TypeScript 文件并返回 stdout             |
| `run_module`       | 作为子进程运行 Osmedeus 模块                           |
| `run_flow`         | 作为子进程运行 Osmedeus flow                           |

### 自定义工具

使用显式 schema 和 JavaScript 处理器定义自定义工具：

```yaml
agent_tools:
  - preset: bash
  - name: check_port
    description: "检查主机上的端口是否开放"
    parameters:
      type: object
      properties:
        host:
          type: string
          description: "目标主机名或 IP"
        port:
          type: integer
          description: "要检查的端口号"
      required: ["host", "port"]
    handler: |
      exec("nc -zv -w3 " + args.host + " " + args.port)
```

`handler` 是一个 JavaScript 表达式。解析后的工具调用参数可作为 `args` 对象使用。

### 多目标执行

使用 `queries` 让代理按顺序执行多个目标。每个目标依次执行，所有结果都会被收集：

```yaml
- name: full-recon
  type: agent
  queries:
    - "发现 {{Target}} 的所有子域名"
    - "识别已发现子域名上运行的 Web 服务"
    - "检查每个服务的常见错误配置"
  system_prompt: "你是一名彻底的审计员。"
  max_iterations: 20
  agent_tools:
    - preset: bash
    - preset: read_file
    - preset: save_content
  exports:
    all_results: "{{agent_goal_results}}"
    final_output: "{{agent_content}}"
```

`agent_goal_results` 导出包含所有目标的结果，格式为 JSON 数组。

### 规划阶段

在主执行循环之前添加规划阶段。代理首先生成计划，然后执行：

```yaml
- name: planned-scan
  type: agent
  query: "对 {{Target}} 执行全面的安全评估"
  plan_prompt: |
    为评估 {{Target}} 创建逐步计划。
    考虑：子域名枚举、服务检测、漏洞扫描。
  plan_max_tokens: 1000
  max_iterations: 25
  agent_tools:
    - preset: bash
    - preset: read_file
    - preset: save_content
  exports:
    plan: "{{agent_plan}}"
    results: "{{agent_content}}"
```

| 字段             | 类型   | 描述                                                               |
| ---------------- | ------ | ------------------------------------------------------------------ |
| `plan_prompt`    | string | 规划阶段的提示（在主循环前触发计划生成）                           |
| `plan_max_tokens`| int    | 计划响应的最大 token 数                                            |

### 内存管理

控制长时间运行代理的对话上下文大小：

```yaml
- name: long-running-agent
  type: agent
  query: "对 {{Target}} 执行深度侦察"
  max_iterations: 50
  memory:
    max_messages: 30
    summarize_on_truncate: true
    persist_path: "{{Output}}/agent/conversation.json"
    resume_path: "{{Output}}/agent/conversation.json"
  agent_tools:
    - preset: bash
    - preset: read_file
    - preset: save_content
```

| 字段                   | 类型   | 默认值       | 描述                                                               |
| ---------------------- | ------ | ------------ | ------------------------------------------------------------------ |
| `max_messages`         | int    | 0（无限制）  | 滑动窗口大小；超出时丢弃最旧的非系统消息                           |
| `summarize_on_truncate`| bool   | false        | 使用 LLM 总结丢弃的消息，而不是静默丢弃                            |
| `persist_path`         | string | —            | 完成后保存对话 JSON                                                |
| `resume_path`          | string | —            | 启动时加载之前的对话（支持跨运行继续）                             |

### 模型偏好

指定按顺序尝试的首选模型。如果都不可用，则回退到默认供应商配置：

```yaml
- name: smart-agent
  type: agent
  query: "分析 {{Target}} 的复杂目标架构"
  max_iterations: 10
  models:
    - claude-3-opus
    - gpt-4
    - claude-3-sonnet
  agent_tools:
    - preset: bash
```

### 结构化输出（Agent）

使用 `output_schema` 对代理的最终输出强制执行 JSON schema：

```yaml
- name: structured-agent
  type: agent
  query: "查找 {{Target}} 上的所有开放端口和服务"
  max_iterations: 15
  output_schema: '{"type":"object","properties":{"ports":{"type":"array","items":{"type":"object","properties":{"port":{"type":"integer"},"service":{"type":"string"},"version":{"type":"string"}}}},"summary":{"type":"string"}},"required":["ports","summary"]}'
  agent_tools:
    - preset: bash
    - preset: save_content
  exports:
    structured_results: "{{agent_content}}"
```

通过 OpenAI 的 `response_format` 参数在最后一次迭代时强制执行 schema。

### 子代理

定义内联子代理，父代理可以通过自动生成的 `spawn_agent` 工具生成它们：

```yaml
- name: coordinator
  type: agent
  query: "使用每个阶段的专业子代理评估 {{Target}}。"
  system_prompt: "你是一个协调代理。将任务委托给专业子代理。"
  max_iterations: 10
  max_agent_depth: 3
  agent_tools:
    - preset: read_file
    - preset: save_content
  sub_agents:
    - name: subdomain-scanner
      description: "发现目标域名的子域名"
      system_prompt: "你是一个子域名枚举专家。"
      max_iterations: 10
      agent_tools:
        - preset: bash
        - preset: save_content

    - name: vuln-checker
      description: "检查已发现服务上的漏洞"
      system_prompt: "你是一个漏洞评估专家。"
      max_iterations: 10
      agent_tools:
        - preset: bash
        - preset: read_file
      output_schema: '{"type":"object","properties":{"vulnerabilities":{"type":"array"}}}'
  exports:
    assessment: "{{agent_content}}"
```

当定义了 `sub_agents` 时，会自动添加一个 `spawn_agent` 工具，参数包括：

* `agent` — 要生成的子代理名称（来自已定义的列表）
* `query` — 要委托的任务

子代理支持递归嵌套（子代理可以定义自己的 `sub_agents`）。使用 `max_agent_depth` 控制嵌套深度（默认：3）。

### 停止条件

一个 JavaScript 表达式，在每次迭代后评估。如果返回 `true`，代理停止：

```yaml
- name: targeted-scan
  type: agent
  query: "查找 {{Target}} 的管理面板"
  max_iterations: 20
  stop_condition: 'agent_content.includes("admin") && iteration > 3'
  agent_tools:
    - preset: bash
    - preset: read_file
```

表达式中可用的变量：`agent_content`（当前响应文本）、`iteration`（当前迭代次数）。

### 工具追踪钩子

在每次工具调用前后执行的 JavaScript 表达式，用于日志记录或调试：

```yaml
- name: traced-agent
  type: agent
  query: "扫描 {{Target}}"
  max_iterations: 10
  on_tool_start: 'log_info("调用工具: " + tool_name + " 参数: " + tool_args)'
  on_tool_end: 'log_info("工具 " + tool_name + " 返回: " + tool_result.substring(0, 200))'
  agent_tools:
    - preset: bash
    - preset: read_file
```

| 钩子            | 可用变量                               |
| --------------- | -------------------------------------- |
| `on_tool_start` | `tool_name`, `tool_args`               |
| `on_tool_end`   | `tool_name`, `tool_args`, `tool_result`|

### 并行工具调用

默认情况下，代理允许 LLM 并行进行多个工具调用。禁用此功能以进行顺序执行：

```yaml
-
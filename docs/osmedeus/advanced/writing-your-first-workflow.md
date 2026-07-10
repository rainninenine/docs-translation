> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 编写你的第一个工作流

> 基于 YAML 的工作流定义与执行

<Callout icon="lightbulb-message" color="#FFC107" iconType="regular">
  你可以在[工作流测试套件](https://github.com/j3ssie/osmedeus/tree/main/test/testdata/workflows)中找到所有可用工作流类型和步骤的全面示例。

  你也可以使用 [osmedeus/osmedeus-skills](https://github.com/osmedeus/osmedeus-skills) 提供的技能。这些技能可以帮助你的 AI 代理自动为你生成工作流。
</Callout>

本指南将引导你从基本概念到高级模式，逐步创建 Osmedeus 工作流。

## 工作流类型

Osmedeus 支持两种工作流类型：

| 类型     | 用途                          |
| -------- | -------------------------------- |
| `module` | 包含步骤的单一执行单元 |
| `flow`   | 编排多个模块    |

## 基本结构

### 模块工作流

```yaml
name: my-first-workflow
kind: module
description: 一个简单的工作流示例
tags: example,tutorial

params:
  - name: custom_param
    required: false
    default: "default_value"

steps:
  - name: hello-world
    type: bash
    command: echo "Hello, {{Target}}!"
```

### 流程工作流

```yaml
name: my-flow
kind: flow
description: 编排多个模块

modules:
  - name: subdomain-enum
    path: modules/subdomain-enum.yaml

  - name: port-scan
    path: modules/port-scan.yaml
    depends_on:
      - subdomain-enum
```

## 步骤类型

### bash - 执行 Shell 命令

```yaml
# 单条命令
- name: simple-command
  type: bash
  command: echo "Hello {{Target}}"

# 多条顺序命令
- name: multiple-commands
  type: bash
  commands:
    - mkdir -p {{Output}}/results
    - echo "{{Target}}" > {{Output}}/target.txt

# 并行命令
- name: parallel-commands
  type: bash
  parallel_commands:
    - 'curl -s https://api1.example.com'
    - 'curl -s https://api2.example.com'

# 结构化参数（适用于 nuclei 等工具）
- name: nuclei-scan
  type: bash
  command: nuclei
  speed_args: '-c {{threads}}'
  config_args: '-t /templates'
  input_args: '-l {{Output}}/urls.txt'
  output_args: '-o {{Output}}/nuclei.json'
```

### function - JavaScript 工具函数

```yaml
# 单个函数
- name: check-file
  type: function
  function: 'fileExists("{{Output}}/results.txt")'

# 多个函数
- name: process-results
  type: function
  functions:
    - 'log_info("Processing results...")'
    - 'var count = fileLength("{{Output}}/results.txt")'
    - 'log_info("Found " + count + " results")'

# 并行函数
- name: parallel-logging
  type: function
  parallel_functions:
    - 'log_info("Task A")'
    - 'log_info("Task B")'
```

### parallel-steps - 并发运行步骤

```yaml
- name: parallel-recon
  type: parallel-steps
  parallel_steps:
    - name: subfinder
      type: bash
      command: subfinder -d {{Target}} -o {{Output}}/subfinder.txt

    - name: assetfinder
      type: bash
      command: assetfinder {{Target}} > {{Output}}/assetfinder.txt

    - name: amass
      type: bash
      command: amass enum -passive -d {{Target}} -o {{Output}}/amass.txt
```

### foreach - 遍历输入

```yaml
- name: scan-subdomains
  type: foreach
  input: "{{Output}}/subdomains.txt"
  variable: subdomain
  threads: 10

  step:
    name: httpx-probe
    type: bash
    command: 'httpx -u [[subdomain]] -silent'
```

**注意：** 在 foreach 循环内部使用 `[[variable]]` 语法以避免模板冲突。

### http - 发起 HTTP 请求

```yaml
- name: api-call
  type: http
  url: "https://api.example.com/scan"
  method: POST
  headers:
    Content-Type: application/json
    Authorization: "Bearer {{api_token}}"
  request_body: |
    {
      "target": "{{Target}}",
      "options": {"deep": true}
    }
  exports:
    response_data: "{{response.body}}"
```

### llm - AI 驱动分析

```yaml
- name: ai-analysis
  type: llm
  messages:
    - role: system
      content: "你是一名安全分析师。"
    - role: user
      content: "分析 {{Target}} 的扫描结果"
  llm_config:
    model: gpt-4
    max_tokens: 1000
    temperature: 0.7
  exports:
    analysis: "{{llm_step_content}}"
```

### agent - 代理式 LLM 执行

```yaml
- name: analyze-target
  type: agent
  query: "枚举 {{Target}} 的子域名并总结发现。"
  system_prompt: "你是一名安全侦察代理。"
  max_iterations: 10
  agent_tools:
    - preset: bash
    - preset: read_file
    - preset: save_content
  memory:
    max_messages: 30
    persist_path: "{{Output}}/agent/conversation.json"
  exports:
    findings: "{{agent_content}}"
```

agent 步骤类型创建一个自主工具调用循环。代理接收任务，规划方法，迭代调用工具，并生成最终答案。关键字段：

* `query` — 代理的任务提示
* `max_iterations` — 最大工具调用循环迭代次数（必填）
* `agent_tools` — 预设或自定义工具列表（例如 `bash`、`read_file`、`save_content`、`grep_regex`、`http_get`）
* `memory` — 对话记忆配置
* `exports` — 使用 `{{agent_content}}` 获取最终响应文本

完整参考请参见 [步骤类型 - agent](../workflows/step-types#agent)。

## 模板变量

### 内置变量

| 变量                        | 描述                                     |
| --------------------------- | ----------------------------------------------- |
| `{{Target}}`                | 当前目标                                  |
| `{{Output}}`                | 本次运行的输出目录                   |
| `{{BaseFolder}}`            | Osmedeus 安装目录                 |
| `{{Binaries}}`              | 二进制工具目录                          |
| `{{Data}}`                  | 数据目录（字典等）                |
| `{{Workflows}}`             | 工作流目录                             |
| `{{Workspaces}}`            | 项目空间目录                            |
| `{{threads}}`               | 基于策略的线程数                    |
| `{{Version}}`               | Osmedeus 版本                                |
| `{{PlatformOS}}`            | 操作系统（`linux`、`darwin`、`windows`） |
| `{{PlatformArch}}`          | CPU 架构（`amd64`、`arm64`） |
| `{{PlatformInDocker}}`      | 如果在 Docker 中运行则为 `"true"`                   |
| `{{PlatformInKubernetes}}`  | 如果在 Kubernetes 中运行则为 `"true"`               |
| `{{PlatformCloudProvider}}` | 云供应商（`aws`、`gcp`、`azure`、`local`） |

### Foreach 循环变量

在 foreach 循环内部使用双括号 `[[variable]]`：

```yaml
- name: process-items
  type: foreach
  input: "{{Output}}/items.txt"
  variable: item
  step:
    type: bash
    command: 'process [[item]] --output {{Output}}/[[item]].json'
```

## 导出与变量传递

使用 exports 在步骤之间传递数据：

```yaml
- name: count-results
  type: bash
  command: wc -l {{Output}}/results.txt | awk '{print $1}'
  exports:
    result_count: "output"  # 特殊：捕获标准输出

- name: log-count
  type: function
  function: 'log_info("Found {{result_count}} results")'
```

## 决策路由

根据条件分支工作流执行。决策支持两种模式：**switch/case**（精确字符串匹配）和 **conditions**（布尔表达式）。

### Switch/Case 模式

将变量值与精确字符串匹配：

```yaml
- name: detect-type
  type: bash
  command: 'detect-target-type {{Target}}'
  exports:
    target_type: "output"

  decision:
    switch: "{{target_type}}"
    cases:
      "domain":
        goto: subdomain-enum
      "ip":
        goto: port-scan
      "url":
        goto: web-scan
    default:
      goto: generic-recon

- name: subdomain-enum
  type: bash
  command: subfinder -d {{Target}}

- name: port-scan
  type: bash
  command: nmap {{Target}}

# 使用 goto: _end 提前终止工作流
```

### 分支中的内联操作

每个分支可以运行内联命令或函数，替代（或附加于）`goto`。当与 `goto` 结合时，内联操作先执行，然后跳转。

**可用的分支字段：**

| 字段        | 类型      | 描述                                    |
| ----------- | --------- | ---------------------------------------------- |
| `goto`      | string    | 按名称跳转到步骤，或使用 `_end` 终止 |
| `command`   | string    | 内联运行单条 bash 命令               |
| `commands`  | string\[] | 顺序运行多条 bash 命令         |
| `function`  | string    | 执行单个工具函数              |
| `functions` | string\[] | 顺序执行多个工具函数 |

```yaml
- name: setup-scan
  type: bash
  command: echo "Setting up scan"
  exports:
    setup_mode: "{{scan_mode}}"
  decision:
    switch: "{{setup_mode}}"
    cases:
      "quick":
        functions:
          - "log_info('Configuring quick scan')"
          - "log_info('Quick scan configured')"
      "deep":
        functions:
          - "log_info('Configuring deep scan')"
          - "log_info('Deep scan configured')"
        goto: deep-scan-step
      "custom":
        command: echo "Running custom setup"
      "multi":
        commands:
          - echo "Step 1: prepare environment"
          - echo "Step 2: configure tools"
    default:
      function: "log_info('Using default scan mode')"
```

### 条件模式

使用 JavaScript 布尔表达式实现更灵活的路由。所有匹配的条件都会执行（无短路），最后一个匹配的 `goto` 生效。

```yaml
- name: check-results
  type: bash
  command: echo "Checking results"
  decision:
    conditions:
      - if: "{{enable_extra}} && {{target}} != ''"
        function: "log_info('Extra scanning enabled')"
        goto: extra-scan
      - if: "file_length('{{Output}}/results.txt') > 100"
        functions:
          - "log_info('Large result set detected')"
          - "log_info('Switching to batch processing')"
        goto: batch-process
      - if: "file_exists('{{Output}}/errors.log')"
        command: echo "Errors detected, reviewing..."
```

条件支持模板变量、函数调用和标准 JavaScript 运算符。

## 处理器（on_success / on_error）

```yaml
- name: critical-scan
  type: bash
  command: 'nuclei -u {{Target}}'

  on_success:
    - action: log
      message: "Scan completed for {{Target}}"
    - action: notify
      notify: "Scan finished: {{Target}}"
    - action: export
      name: scan_status
      value: "success"

  on_error:
    - action: log
      message: "Scan failed for {{Target}}"
    - action: continue  # 出错后继续执行
    # 或者：action: abort 停止工作流
```

## 工作流钩子

钩子允许你在主工作流执行前后运行步骤。用于设置、清理、通知或结果后处理。

```yaml
name: recon-with-hooks
kind: module
description: 带设置和清理钩子的侦察工作流

hooks:
  pre_scan_steps:
    - name: setup-workspace
      type: bash
      commands:
        - mkdir -p {{Output}}/results
        - echo "Scan started at $(date)" > {{Output}}/scan.log

    - name: notify-start
      type: function
      function: |
        generate_event("{{Workspace}}", "scan.started", "workflow", "status", "{{Target}}")

  post_scan_steps:
    - name: generate-report
      type: function
      function: |
        convert_sarif_to_markdown("{{Output}}/results.sarif", "{{Output}}/report.md")

    - name: notify-complete
      type: function
      function: |
        generate_event("{{Workspace}}", "scan.completed", "workflow", "status", "{{Target}}")

    - name: cleanup-temp
      type: bash
      command: rm -rf {{Output}}/tmp

steps:
  - name: run-scan
    type: bash
    command: nuclei -u {{Target}} -sarif-export {{Output}}/results.sarif
```

### 钩子执行顺序

```
pre_scan_steps  →  steps（主工作流）  →  post_scan_steps
```

* **pre_scan_steps** 在任何主步骤之前运行
* **post_scan_steps** 在所有主步骤完成后运行
* 两者都支持所有步骤类型（bash、function、parallel-steps、foreach 等）
* 钩子步骤可以访问与主步骤相同的模板变量

### 流程级钩子

钩子也适用于流程。它们在整个流程周围运行一次，而不是每个模块：

```yaml
name: full-recon
kind: flow
description: 带钩子的完整侦察流程

hooks:
  pre_scan_steps:
    - name: pre-flight-check
      type: function
      function: 'log_info("Starting flow for " + "{{Target}}")'

  post_scan_steps:
    - name: aggregate-results
      type: bash
      command: cat {{Output}}/*/findings.txt | sort -u > {{Output}}/all-findings.txt

modules:
  - name: subdomain-enum
    path: modules/subdomain-enum.yaml
  - name: port-scan
    path: modules/port-scan.yaml
```

## Runner 配置

### Host Runner（默认）

```yaml
runner: host  # 本地运行，此为默认值
```

### Docker Runner

```yaml
runner: docker
runner_config:
  image: ubuntu:22.04
  env:
    API_KEY: "{{api_key}}"
  volumes:
    - "{{Output}}:/output"
  network: host
  persistent: false
```

### SSH Runner

```yaml
runner: ssh
runner_config:
  host: 192.168.1.100
  port: 22
  user: scanner
  key_file: ~/.ssh/id_rsa
```

### 按步骤覆盖 Runner

```yaml
- name: docker-step
  type: bash
  step_runner: docker
  step_runner_config:
    image: projectdiscovery/nuclei:latest
  command: 'nuclei -u {{Target}}'
```

## 完整示例

```yaml
name: basic-recon
kind: module
description: 基础侦察工作流
tags: recon,subdomain,fast

params:
  - name: threads
    default: "10"

steps:
  - name: setup
    type: bash
    commands:
      - mkdir -p {{Output}}/subdomains
      - mkdir -p {{Output}}/urls

  - name: passive-enum
    type: parallel-steps
    parallel_steps:
      - name: subfinder
        type: bash
        command: subfinder -d {{Target}} -silent -o {{Output}}/subdomains/subfinder.txt

      - name: assetfinder
        type: bash
        command: assetfinder --subs-only {{Target}} > {{Output}}/subdomains/assetfinder.txt

  - name: merge-results
    type: bash
    commands:
      - cat {{Output}}/subdomains/*.txt | sort -u > {{Output}}/subdomains.txt
    exports:
      subdomain_count: "output"

  - name: probe-http
    type: foreach
    input: "{{Output}}/subdomains.txt"
    variable: sub
    threads: "{{threads}}"
    step:
      name: httpx
      type: bash
      command: 'httpx -u [[sub]] -silent >> {{Output}}/urls/live.txt'

  - name: summary
    type: function
    functions:
      - 'var total = fileLength("{{Output}}/subdomains.txt")'
      - 'var live = fileLength("{{Output}}/urls/live.txt")'
      - 'log_info("Found " + total + " subdomains, " + live + " live hosts")'
```

## 运行你的工作流

```bash
# 运行模块工作流
osmedeus run -m basic-recon -t example.com

# 使用自定义参数运行
osmedeus run -m basic-recon -t example.com --params 'threads=20'

# 空运行（预览而不执行）
osmedeus run -m basic-recon -t example.com --dry-run

# 运行并输出详细信息
osmedeus run -m basic-recon -t example.com -v
```

## 下一步

* 查看 [CLI 参考](cli-references.md) 获取所有命令选项
* 查看 [扩展 Osmedeus](extending-osmedeus.md) 添加自定义步骤类型
* 查看 [高级配置](advanced-configuration.md) 了解 API 密钥和存储设置
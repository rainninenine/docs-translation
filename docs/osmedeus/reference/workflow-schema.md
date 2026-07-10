> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 工作流模式参考

> Osmedeus 工作流的完整 YAML 模式参考

# 工作流模式参考

本文档提供 Osmedeus 工作流 YAML 模式的完整参考。

## 顶层字段

所有工作流共享以下通用顶层字段：

```yaml
kind: module | flow | fragment     # 必需：工作流类型
name: workflow-name               # 必需：唯一标识符
description: "描述文本"           # 可选：人类可读的描述
tags: "recon, fast"               # 可选：逗号分隔的标签

extends: parent-workflow          # 可选：要继承的父工作流
override:                         # 可选：覆盖父工作流的节
  params: {}
  steps: {}
  modules: {}

params:                           # 可选：输入参数
  - name: param_name
    default: "value"
    required: false

triggers:                          # 可选：自动触发器
  - name: trigger-name
    on: cron | event | watch | manual
    enabled: true

dependencies:                     # 可选：外部工具依赖
  binaries:
    - nuclei
    - httpx
```

## 工作流类型

| 类型       | 用途                  | 包含内容        |
| ---------- | ------------------------ | --------------- |
| `module`   | 单一执行单元    | `steps` 数组   |
| `flow`     | 编排模块      | `modules` 数组 |
| `fragment` | 可复用的步骤集合 | `steps` 数组   |

## 工作流继承

工作流可以使用 `extends` 和 `override` 字段扩展父工作流。

### Extends 字段

```yaml
# 按名称（在工作流目录中搜索）
extends: parent-workflow-name

# 按相对路径
extends: ./parent.yaml

# 按工作流目录中的路径
extends: modules/base-module.yaml
```

### Override 模式

```yaml
override:
  # 覆盖特定参数属性
  params:
    param_name:
      default: "new-value"
      type: "string"
      required: true
      generator: "generator_func"

  # 覆盖步骤（仅模块工作流）
  steps:
    mode: replace | prepend | append | merge
    steps:                    # 要添加/替换的步骤
      - name: new-step
        type: bash
        command: "..."
    remove:                   # 要移除的步骤名称（仅 merge 模式）
      - step-to-remove
    replace:                  # 按名称替换的步骤（仅 merge 模式）
      - name: existing-step
        type: bash
        command: "new-command"

  # 覆盖模块（仅流程工作流）
  modules:
    mode: replace | prepend | append | merge
    modules:                  # 要添加/替换的模块
      - name: new-module
        path: modules/new.yaml
    remove:                   # 要移除的模块名称（仅 merge 模式）
      - module-to-remove
    replace:                  # 按名称替换的模块（仅 merge 模式）
      - name: existing-module
        path: modules/replacement.yaml

  # 完全替换触发器
  triggers:
    - name: new-trigger
      on: cron
      schedule: "0 * * * *"

  # 与父依赖合并
  dependencies:
    commands:
      - new-tool
    files:
      - /path/to/file

  # 覆盖偏好设置（子项覆盖父项）
  preferences:
    disable_notifications: true
    silent: false

  # 覆盖 Runner 配置
  runner_config:
    image: "custom-image:latest"
    env:
      NEW_VAR: "value"

  # 覆盖 Runner 类型
  runner: docker
```

### 覆盖模式

| 模式      | 描述                                                   |
| --------- | ------------------------------------------------------------- |
| `replace` | 用子项完全替换父项              |
| `prepend` | 在父项之前添加子项                           |
| `append`  | 在父项之后添加子项（默认）                  |
| `merge`   | 按名称匹配：替换匹配项，追加新项，移除指定项 |

### 继承示例

```yaml
# 父级：base-enum.yaml
kind: module
name: base-enum
params:
  - name: threads
    default: 10
steps:
  - name: subfinder
    type: bash
    command: "subfinder -d {{Target}}"

# 子级：fast-enum.yaml
kind: module
name: fast-enum
extends: base-enum

override:
  params:
    threads:
      default: 50
  steps:
    mode: append
    steps:
      - name: additional-tool
        type: bash
        command: "extra-tool -t {{Target}}"
```

## 模块特定字段

模块包含按顺序或基于依赖关系执行的步骤。

```yaml
kind: module
name: my-module

# Runner 配置（可选）
runner: host | docker | ssh
runner_config:
  # Docker 设置
  image: "ubuntu:latest"
  env:
    KEY: "value"
  volumes:
    - "/host/path:/container/path"
  network: "host"
  persistent: false

  # SSH 设置
  host: "remote.example.com"
  port: 22
  user: "username"
  key_file: "~/.ssh/id_rsa"
  workdir: "/tmp"

# 片段包含
includes:
  - path: fragments/common-setup.yaml
    fragment_name: setup           # 片段步骤引用的名称
    params:
      custom_var: "{{Target}}"
    position: prepend | append

# 步骤数组
steps:
  - name: step-name
    type: bash
    command: "echo hello"
```

## 流程特定字段

流程编排多个模块及其依赖关系。

```yaml
kind: flow
name: my-flow

modules:
  - name: first-module
    path: modules/first.yaml
    params:
      threads: "10"

  - name: second-module
    path: modules/second.yaml
    depends_on:
      - first-module
    condition: "fileLength('{{Output}}/subdomains.txt') > 0"
    on_success:
      - action: log
        message: "模块完成"
    on_error:
      - action: notify
        notify: "模块失败"
    decision:
      switch: "{{status}}"
      cases:
        "critical": { goto: alert-module }
        "none": { goto: _end }
      default: { goto: next-module }
```

## 片段特定字段

片段是可复用的步骤集合，可包含在模块中。

```yaml
kind: fragment
name: common-subdomain-enum

params:
  - name: wordlist
    default: "{{Data}}/wordlists/subdomains.txt"

steps:
  - name: subfinder
    type: bash
    command: "subfinder -d {{Target}} -o {{Output}}/subfinder.txt"

  - name: amass
    type: bash
    command: "amass enum -d {{Target}} -o {{Output}}/amass.txt"
```

## 步骤类型

### 步骤类型：bash

在本地执行 Shell 命令。

```yaml
- name: run-nuclei
  type: bash
  command: "nuclei -l {{Output}}/urls.txt -o {{Output}}/nuclei.json"
  timeout: 1h

# 多个顺序命令
- name: multi-commands
  type: bash
  commands:
    - "echo 'Step 1'"
    - "echo 'Step 2'"

# 并行命令
- name: parallel-scans
  type: bash
  parallel_commands:
    - "nuclei -l urls.txt -t cves/"
    - "nuclei -l urls.txt -t exposed-panels/"
```

### 步骤类型：function

通过 Goja JavaScript 运行时执行实用函数。

```yaml
- name: check-results
  type: function
  function: "fileExists('{{Output}}/results.txt')"
  exports:
    has_results: "{{_result}}"

# 多个函数
- name: process-data
  type: function
  functions:
    - "log_info('Processing started')"
    - "sortUnix('{{Output}}/urls.txt')"
    - "remove_blank_lines('{{Output}}/urls.txt')"
```

### 步骤类型：parallel-steps

并发执行多个步骤。

```yaml
- name: parallel-enum
  type: parallel-steps
  parallel_steps:
    - name: subfinder
      type: bash
      command: "subfinder -d {{Target}} -o {{Output}}/subfinder.txt"
    - name: amass
      type: bash
      command: "amass enum -d {{Target}} -o {{Output}}/amass.txt"
```

### 步骤类型：foreach

对输入行进行迭代并并行处理。

```yaml
- name: scan-subdomains
  type: foreach
  input: "{{Output}}/subdomains.txt"
  variable: subdomain
  threads: 10
  step:
    name: httpx-probe
    type: bash
    command: "httpx -u [[subdomain]] >> {{Output}}/httpx.txt"
```

### 步骤类型：remote-bash

在 Docker 容器中或通过 SSH 执行命令。

```yaml
# Docker 执行
- name: docker-scan
  type: remote-bash
  step_runner: docker
  step_runner_config:
    image: "projectdiscovery/nuclei:latest"
    volumes:
      - "{{Output}}:/output"
  command: "nuclei -l /output/urls.txt -o /output/nuclei.json"
  step_remote_file: "/output/nuclei.json"
  host_output_file: "{{Output}}/nuclei.json"

# SSH 执行
- name: ssh-scan
  type: remote-bash
  step_runner: ssh
  step_runner_config:
    host: "scan-server.example.com"
    user: "scanner"
    key_file: "~/.ssh/id_rsa"
  command: "nuclei -l /tmp/urls.txt"
```

### 步骤类型：http

发起 HTTP 请求并捕获响应。

```yaml
- name: api-call
  type: http
  url: "https://api.example.com/scan"
  method: POST
  headers:
    Authorization: "Bearer {{api_token}}"
    Content-Type: "application/json"
  request_body: '{"target": "{{Target}}"}'
  exports:
    response: "{{_result}}"
```

### 步骤类型：llm

与 LLM API（兼容 OpenAI）交互。

```yaml
- name: analyze-vulns
  type: llm
  messages:
    - role: system
      content: "你是一名安全分析师。"
    - role: user
      content: "分析以下发现：{{findings}}"
  llm_config:
    model: "gpt-4"
    temperature: 0.7
    max_tokens: 2000
  exports:
    analysis: "{{_result}}"

# 嵌入
- name: embed-text
  type: llm
  is_embedding: true
  embedding_input:
    - "要嵌入的文本"
  exports:
    embedding: "{{_result}}"
```

### 步骤类型：fragment-step

内联执行片段，支持可选覆盖。

```yaml
# 在模块中包含片段
includes:
  - path: fragments/subdomain-enum.yaml
    fragment_name: subdomain-enum

steps:
  - name: run-subdomain-enum
    type: fragment-step
    fragment_name: subdomain-enum
    override:
      threads: "20"                    # 覆盖步骤字段
      wordlist: "{{custom_wordlist}}"  # 覆盖模板变量
```

## 通用步骤字段

以下字段适用于所有步骤类型：

```yaml
- name: step-name                    # 必需：唯一步骤标识符
  type: bash                         # 必需：步骤类型

  depends_on:                        # 可选：步骤依赖（DAG）
    - previous-step

  pre_condition: "fileExists('{{file}}')"  # 可选：条件为假时跳过

  timeout: 30m                       # 可选：步骤超时（例如 30, 30s, 30m, 1h, 1d）

  log: "{{Output}}/step.log"         # 可选：日志文件路径

  exports:                           # 可选：导出变量
    result_count: "{{_result}}"

  on_success:                        # 可选：成功处理程序
    - action: log
      message: "步骤完成"

  on_error:                          # 可选：错误处理程序
    - action: abort
      message: "严重故障"

  decision:                          # 可选：条件路由
    switch: "{{status}}"
    cases:
      "critical": { goto: alert-step }
    default: { goto: next-step }
```

## 决策路由

步骤支持使用 switch/case 语法进行条件分支：

```yaml
decision:
  switch: "{{variable}}"
  cases:
    "value1": { goto: step-a }
    "value2": { goto: step-b }
  default: { goto: fallback }
```

使用 `goto: _end` 终止工作流。

## 参数

参数定义工作流输入及验证：

```yaml
params:
  - name: threads
    type: number
    default: 10
    required: false
    description: "线程数"

  - name: wordlist
    type: file
    default: "{{Data}}/wordlists/common.txt"
    required: false

  - name: target_type
    type: string
    required: true
```

## 触发器

触发器定义自动执行：

```yaml
triggers:
  # Cron 触发器
  - name: daily-scan
    on: cron
    schedule: "0 2 * * *"
    enabled: true

  # 事件触发器
  - name: on-new-asset
    on: event
    topic: "assets.new"
    filter:
      asset_type: "subdomain"
    enabled: true

  # 文件监视触发器
  - name: watch-targets
    on: watch
    paths:
      - "{{Data}}/targets.txt"
    enabled: true

  # 手动（默认）
  - name: manual
    on: manual
    enabled: true
```

## 完整示例

### 模块示例

```yaml
kind: module
name: subdomain-enum
description: "枚举目标域的子域"
tags: "recon, subdomain"

params:
  - name: threads
    default: 10
  - name: wordlist
    default: "{{Data}}/wordlists/subdomains.txt"

steps:
  - name: subfinder
    type: bash
    command: "subfinder -d {{Target}} -all -o {{Output}}/subfinder.txt"
    timeout: 30m

  - name: merge-results
    type: function
    depends_on:
      - subfinder
    functions:
      - "sortUnix('{{Output}}/subfinder.txt', '{{Output}}/subdomains.txt')"
      - "db_total_subdomains('{{Output}}/subdomains.txt')"
```

### 流程示例

```yaml
kind: flow
name: full-recon
description: "完整侦察工作流"
tags: "recon, full"

modules:
  - name: subdomain-enum
    path: modules/subdomain-enum.yaml
    params:
      threads: "20"

  - name: port-scan
    path: modules/port-scan.yaml
    depends_on:
      - subdomain-enum
    condition: "fileLength('{{Output}}/subdomains.txt') > 0"

  - name: vuln-scan
    path: modules/vuln-scan.yaml
    depends_on:
      - port-scan
```

### 片段示例

```yaml
kind: fragment
name: common-cleanup
description: "通用清理步骤"

steps:
  - name: deduplicate
    type: function
    function: "sortUnix('{{input_file}}')"

  - name: remove-blanks
    type: function
    function: "remove_blank_lines('{{input_file}}')"
```

### 使用片段的模块

```yaml
kind: module
name: subdomain-scan

includes:
  - path: fragments/common-cleanup.yaml
    fragment_name: cleanup

steps:
  - name: subfinder
    type: bash
    command: "subfinder -d {{Target}} -o {{Output}}/subs-raw.txt"

  - name: cleanup-results
    type: fragment-step
    fragment_name: cleanup
    override:
      input_file: "{{Output}}/subs-raw.txt"
```
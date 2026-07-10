> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 控制流

> 条件、路由与错误处理

通过条件、处理器和决策路由控制执行。

## 前置条件

如果条件为假，则跳过步骤。

```yaml
- name: nuclei-scan
  type: bash
  pre_condition: 'fileLength("{{Output}}/live.txt") > 0'
  command: nuclei -l {{Output}}/live.txt -o {{Output}}/vulns.txt
```

### 常用条件

```yaml
# 文件存在
pre_condition: 'fileExists("{{Output}}/targets.txt")'

# 文件有内容
pre_condition: 'fileLength("{{Output}}/hosts.txt") > 0'

# 检查导出值
pre_condition: '{{host_count}} > 10'

# 检查参数
pre_condition: '{{enable_scan}} == "true"'

# 组合条件
pre_condition: 'fileExists("{{Output}}/subs.txt") && {{threads}} > 0'
```

### 条件函数

| 函数 | 描述 |
| --- | --- |
| `fileExists(path)` | 如果文件存在则为真 |
| `fileLength(path)` | 非空行数 |
| `dirLength(path)` | 目录条目数 |
| `isEmpty(str)` | 如果字符串为空则为真 |
| `contains(str, substr)` | 如果字符串包含子串则为真 |

## 步骤依赖（DAG 执行）

步骤可以使用 `depends_on` 字段声明对其他步骤的依赖。这支持：

* 独立步骤的并行执行
* 基于依赖关系的自动排序
* DAG（有向无环图）执行

### 基本依赖

```yaml
steps:
  - name: subfinder
    type: bash
    command: "subfinder -d {{Target}} -o {{Output}}/subfinder.txt"

  - name: amass
    type: bash
    command: "amass enum -d {{Target}} -o {{Output}}/amass.txt"

  - name: merge-results
    type: function
    depends_on:
      - subfinder
      - amass
    functions:
      - "appendFile('{{Output}}/all-subs.txt', '{{Output}}/subfinder.txt')"
      - "appendFile('{{Output}}/all-subs.txt', '{{Output}}/amass.txt')"
```

在此示例中：

* `subfinder` 和 `amass` 并行运行（无依赖）
* `merge-results` 等待两者完成后才执行

### DAG 执行

执行器构建依赖图，并使用拓扑排序（Kahn 算法）确定执行顺序：

```
┌───────────┐     ┌───────────┐
│ subfinder │     │   amass   │
└─────┬─────┘     └─────┬─────┘
      │                 │
      │  ┌──────────────┘
      │  │
      ▼  ▼
┌─────────────┐
│merge-results│
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ http-probe  │
└─────────────┘
```

处于同一“层级”（彼此无依赖）的步骤并发执行。

### 多重依赖

```yaml
- name: final-report
  type: bash
  depends_on:
    - vuln-scan
    - port-scan
    - screenshot
  command: "generate-report {{Output}}"
```

该步骤等待所有列出的依赖成功完成。

### 级联失败

如果某个依赖失败：

1. 依赖步骤被标记为失败（不执行）
2. 所有下游步骤也被标记为失败
3. 独立分支继续执行

```yaml
# 如果 subfinder 失败：
# - merge-results 被跳过
# - amass 继续执行（独立）
```

### 依赖 vs 顺序执行

没有 `depends_on` 时，步骤按顺序依次执行：

```yaml
steps:
  - name: step1     # 先运行
    type: bash
    command: "..."

  - name: step2     # 后运行（等待 step1）
    type: bash
    command: "..."
```

使用 `depends_on` 时，步骤可以并行运行：

```yaml
steps:
  - name: step1     # 先运行（与 step2 并行）
    type: bash
    command: "..."

  - name: step2     # 先运行（与 step1 并行）
    type: bash
    command: "..."

  - name: step3     # 等待两者
    type: bash
    depends_on: [step1, step2]
    command: "..."
```

### Linter 验证

工作流 linter 验证依赖关系：

| 规则 | 描述 |
| --- | --- |
| `invalid-depends-on` | 依赖引用了不存在的步骤 |
| `circular-dependency` | 检测到循环依赖 |

```bash
osmedeus workflow lint my-workflow.yaml
```

### Flow 模块依赖

Flow 也支持模块的 `depends_on`：

```yaml
kind: flow
name: full-pipeline

modules:
  - name: subdomain-enum
    path: modules/subdomain-enum.yaml

  - name: port-scan
    path: modules/port-scan.yaml

  - name: http-probe
    path: modules/http-probe.yaml
    depends_on:
      - subdomain-enum
      - port-scan
```

## 决策路由

根据变量值使用 switch/case 语法路由到不同步骤。

```yaml
steps:
  - name: check-hosts
    type: bash
    command: wc -l < {{Output}}/hosts.txt
    exports:
      count: "{{stdout}}"
    decision:
      switch: "{{count}}"
      cases:
        "0": { goto: no-hosts-found }
      default: { goto: continue-scan }

  - name: no-hosts-found
    type: function
    function: log_warning("No hosts found, skipping scan")
    decision:
      switch: "true"
      cases:
        "true": { goto: _end }    # 特殊：结束工作流

  - name: continue-scan
    type: bash
    command: nuclei -l {{Output}}/hosts.txt -t {{Data}}/templates/
```

### 决策语法

```yaml
decision:
  switch: "{{variable}}"      # 要评估的模板表达式
  cases:                       # 值到动作的映射
    "value1": { goto: step-a }
    "value2": { goto: step-b }
  default: { goto: fallback }  # 可选：当没有 case 匹配时
```

* `switch`：运行时评估的模板变量（精确字符串匹配）
* `cases`：字符串值到 goto 目标的映射
* `default`：当没有 case 匹配时的回退（可选）
* `goto`：目标步骤名称或 `_end` 以终止

### Case 中的内联执行

Case 支持运行命令或函数，而不仅仅是 `goto`：

```yaml
decision:
  switch: "{{target_type}}"
  cases:
    "domain":
      goto: subdomain-enum
      command: "echo 'Processing domain target'"
    "ip":
      goto: port-scan
      commands:
        - "echo 'Processing IP target'"
        - "mkdir -p {{Output}}/ip-results"
    "url":
      function: "log_info('Direct URL target detected')"
      goto: web-scan
```

每个 `DecisionCase` 支持：

| 字段 | 描述 |
| --- | --- |
| `goto` | 目标步骤名称或 `_end` |
| `command` | 要执行的单个命令 |
| `commands` | 要执行的多个命令 |
| `function` | 要执行的单个函数 |
| `functions` | 要执行的多个函数 |

### 基于条件的决策路由

除了 switch/case，决策还支持 `conditions` —— 一个在运行时评估的 JavaScript 表达式数组。所有匹配的条件都会执行（无短路）：

```yaml
- name: smart-routing
  type: function
  function: log_info("Evaluating conditions")
  decision:
    conditions:
      - if: "file_length('{{Output}}/subdomains.txt') > 0"
        goto: process-subdomains

      - if: "file_length('{{Output}}/subdomains.txt') > 100"
        command: "echo 'Large target set detected'"

      - if: "{{enableNmap}} == 'true' && contains('{{Port}}', '-')"
        commands:
          - "nmap -p {{Port}} {{Target}} -oN {{Output}}/nmap.txt"
          - "echo 'Port range scan complete'"

      - if: "{{scan_mode}} == 'aggressive'"
        functions:
          - "log_warning('Aggressive mode enabled')"
          - "log_info('Increasing thread count')"
```

#### 条件字段

| 字段 | 必需 | 描述 |
| --- | --- | --- |
| `if` | 是 | JavaScript 表达式（必须评估为真/假） |
| `goto` | 否 | 目标步骤名称或 `_end` |
| `command` | 否 | 条件为真时执行的单个命令 |
| `commands` | 否 | 条件为真时执行的多个命令 |
| `function` | 否 | 条件为真时执行的单个函数 |
| `functions` | 否 | 条件为真时执行的多个函数 |

!!! note "  数组中的所有匹配条件都会执行——没有短路行为。如果需要独占路由，请改用 switch/case。"

### 特殊 Goto 目标

| 目标 | 描述 |
| --- | --- |
| `_end` | 立即结束工作流 |
| `step-name` | 跳转到指定步骤 |

## 成功处理器

当步骤成功时执行动作。

```yaml
- name: scan
  type: bash
  command: nuclei -l {{Output}}/hosts.txt -o {{Output}}/vulns.txt
  on_success:
    - action: log
      message: "Scan completed successfully"

    - action: export
      key: scan_status
      value: "completed"

    - action: notify
      message: "Vulnerability scan finished for {{target}}"
```

### 可用动作

| 动作 | 描述 | 参数 |
| --- | --- | --- |
| `log` | 记录消息 | `message` |
| `export` | 导出值 | `key`, `value` |
| `run` | 运行命令 | `command` |
| `notify` | 发送通知 | `message` |
| `continue` | 继续执行 | - |

```yaml
on_success:
  - action: log
    message: "Step completed"

  - action: export
    key: result
    value: "success"

  - action: run
    command: echo "Done" >> {{Output}}/log.txt

  - action: notify
    message: "{{target}} scan finished"
```

## 错误处理器

处理步骤失败。

```yaml
- name: risky-scan
  type: bash
  command: aggressive-tool {{target}}
  on_error:
    - action: log
      message: "Scan failed, continuing with fallback"

    - action: continue    # 不停止工作流

    - action: run
      command: fallback-tool {{target}}
```

### 错误动作类型

| 动作 | 描述 |
| --- | --- |
| `log` | 记录错误消息 |
| `abort` | 停止工作流（默认） |
| `continue` | 继续到下一步 |
| `run` | 运行恢复命令 |
| `notify` | 发送错误通知 |

```yaml
on_error:
  - action: abort       # 出错时停止工作流

# 或者

on_error:
  - action: continue    # 忽略错误，继续
```

## 组合示例

```yaml
steps:
  - name: enumerate
    type: bash
    command: subfinder -d {{target}} -o {{Output}}/subs.txt
    exports:
      sub_count: "{{stdout}}"
    on_success:
      - action: log
        message: "Found subdomains"
    on_error:
      - action: log
        message: "Enumeration failed"
      - action: continue

  - name: validate-results
    type: function
    function: fileLength("{{Output}}/subs.txt")
    exports:
      count: "{{result}}"
    decision:
      switch: "{{count}}"
      cases:
        "0": { goto: no-results }
      default: { goto: probe-hosts }

  - name: no-results
    type: function
    function: log_warning("No subdomains found for {{target}}")
    decision:
      switch: "true"
      cases:
        "true": { goto: _end }

  - name: probe-hosts
    type: bash
    pre_condition: 'fileLength("{{Output}}/subs.txt") > 0'
    command: httpx -l {{Output}}/subs.txt -o {{Output}}/live.txt
    on_success:
      - action: export
        key: probe_status
        value: "done"
      - action: notify
        message: "Probing complete for {{target}}"
    on_error:
      - action: log
        message: "HTTP probing failed"
      - action: abort

  - name: screenshot
    type: bash
    pre_condition: 'fileLength("{{Output}}/live.txt") > 0 && {{probe_status}} == "done"'
    command: gowitness file -f {{Output}}/live.txt -P {{Output}}/screenshots
```

## Flow 级别条件

Flow 中的条件模块执行：

```yaml
kind: flow
name: conditional-flow

params:
  - name: target
  - name: enable_active
    default: "false"

modules:
  - name: passive-recon
    path: modules/passive.yaml

  - name: active-scan
    path: modules/active.yaml
    depends_on: [passive-recon]
    condition: '{{enable_active}} == "true"'

  - name: vuln-scan
    path: modules/vuln.yaml
    depends_on: [passive-recon]
    condition: 'fileLength("{{Output}}/live.txt") > 0'
```

## 分支模式

### If-Then-Else

```yaml
- name: check
  type: function
  function: fileLength("{{Output}}/data.txt")
  exports:
    has_data: "{{result}}"
  decision:
    switch: "{{has_data}}"
    cases:
      "0": { goto: handle-empty }
    default: { goto: process-data }

- name: process-data
  type: bash
  command: process {{Output}}/data.txt
  decision:
    switch: "true"
    cases:
      "true": { goto: finalize }

- name: handle-empty
  type: function
  function: log_warning("No data to process")
  decision:
    switch: "true"
    cases:
      "true": { goto: finalize }

- name: finalize
  type: function
  function: log_info("Workflow complete")
```

### 提前退出

```yaml
- name: validate
  type: function
  function: fileExists("{{Output}}/required.txt")
  exports:
    valid: "{{result}}"
  decision:
    switch: "{{valid}}"
    cases:
      "false": { goto: _end }       # 无效则退出
    default: { goto: continue-scan }

- name: continue-scan
  type: bash
  command: scan {{target}}
```

### 带重试的循环

```yaml
- name: attempt-scan
  type: bash
  command: flaky-scanner {{target}}
  exports:
    attempt: "1"
    failed: "false"
  on_error:
    - action: export
      key: failed
      value: "true"
    - action: continue

- name: retry-check
  type: function
  function: log_info("Checking retry status")
  decision:
    switch: "{{failed}}"
    cases:
      "false": { goto: success }
      "true": { goto: check-attempts }
    default: { goto: success }

- name: check-attempts
  type: function
  function: log_info("Attempt {{attempt}}")
  decision:
    switch: "{{attempt}}"
    cases:
      "3": { goto: give-up }
    default: { goto: retry-scan }

- name: retry-scan
  type: bash
  command: flaky-scanner {{target}} --retry
  exports:
    attempt: "{{parseInt({{attempt}}) + 1}}"
  on_error:
    - action: continue
  decision:
    switch: "true"
    cases:
      "true": { goto: retry-check }
```

## 最佳实践

1. **在处理前始终检查文件是否存在**
   ```yaml
   pre_condition: 'fileExists("{{Output}}/input.txt")'
   ```

2. **使用有意义的日志消息**
   ```yaml
   on_success:
     - action: log
       message: "Found {{count}} subdomains for {{target}}"
   ```

3. **优雅地处理错误**
   ```yaml
   on_error:
     - action: log
       message: "Step failed, attempting fallback"
     - action: continue
   ```

4. **对复杂逻辑使用决策路由**
   ```yaml
   decision:
     switch: "{{dataset_size}}"
     cases:
       "large": { goto: large-dataset-handler }
     default: { goto: standard-handler }
   ```

5. **干净地结束工作流**
   ```yaml
   decision:
     switch: "{{fatal_error}}"
     cases:
       "true": { goto: _end }
   ```

## 下一步

* [步骤类型](step-types) - 所有步骤类型
* [变量](variables) - 导出与条件
* [函数参考](../functions/reference) - 条件函数
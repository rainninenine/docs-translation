> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 事件驱动触发器

> 通过事件驱动工作流构建响应式自动化流水线

构建响应式自动化流水线，响应发现结果、串联工作流并与外部系统集成。

## 概述

![触发器事件示例](https://mintcdn.com/osmedeus/umCo_bqCXCReKqLX/images/cli/cli-event-test.png?fit=max&auto=format&n=umCo_bqCXCReKqLX&q=85&s=265d7cb6e3ec6092eb1b0ac46f0e2c94)

事件驱动触发器使工作流能够响应事件自动执行：

* **响应发现结果**：在发现新子域名时立即扫描
* **串联工作流**：连接侦察 → 探测 → 扫描
* **外部集成**：接收来自 GitHub、CI/CD 或自定义工具的 Webhook
* **实时自动化**：在发现结果产生时即时处理

## 事件体系结构

### 事件结构

事件通过系统携带结构化数据：

```go
type Event struct {
    Topic        string    // 类别："assets.new", "vulnerabilities.new"
    ID           string    // 唯一事件标识符（UUID）
    Name         string    // 事件名称："vulnerability.discovered"
    Source       string    // 来源："nuclei", "httpx", "amass"
    Data         string    // JSON 负载
    DataType     string    // 类型："subdomain", "url", "finding"
    Workspace    string    // 项目空间标识符
    RunID        string    // 生成此事件的运行 ID
    WorkflowName string    // 生成此事件的工作流
    Timestamp    time.Time // 事件发生时间
}
```

### 事件流

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   事件源         │     │   事件队列       │     │   调度器         │
│                  │────▶│   (1000 缓冲区)  │────▶│   (过滤 +        │
│  - 工作流        │     │                  │     │    分发)         │
│  - 函数          │     │  背压：          │     │                  │
│  - Webhook       │     │  5秒超时         │     │                  │
└──────────────────┘     └──────────────────┘     └────────┬─────────┘
                                                          │
                        ┌─────────────────────────────────┼─────────────────────────────────┐
                        │                                 │                                 │
                        ▼                                 ▼                                 ▼
               ┌────────────────┐              ┌────────────────┐              ┌────────────────┐
               │  工作流 A      │              │  工作流 B      │              │  工作流 C      │
               │  (主题匹配)    │              │  (已过滤)      │              │  (所有事件)    │
               └────────────────┘              └────────────────┘              └────────────────┘
```

### 背压处理

事件系统防止过载：

| 参数       | 值       | 描述                     |
| ---------- | -------- | ------------------------ |
| 队列大小   | 1000     | 最大缓冲事件数           |
| 超时时间   | 5 秒     | 队列满时的等待时间       |
| 行为       | 丢弃     | 队列持续满时丢弃事件     |

通过指标监控队列健康状态（参见[监控事件](#监控事件)）。

## 发出事件

### 从工作流函数

使用 `generate_event` 函数从工作流发出事件：

```yaml
name: subdomain-discovery
kind: module

steps:
  - name: enumerate
    type: bash
    command: amass enum -d {{target}} -o {{Output}}/subdomains.txt

  - name: emit-discoveries
    type: function
    function: |
      generate_event_from_file("{{Workspace}}", "assets.new", "amass", "subdomain", "{{Output}}/subdomains.txt")
```

#### generate\_event(workspace, topic, source, data\_type, data)

发出单个结构化事件。

```javascript
// 简单字符串数据
generate_event("{{Workspace}}", "assets.new", "httpx", "url", "https://api.example.com")

// 复杂对象数据
generate_event("{{Workspace}}", "vulnerabilities.new", "nuclei", "finding", {
  url: "https://example.com/admin",
  severity: "critical",
  template: "CVE-2024-1234",
  matched: "/admin"
})
```

**参数：**

* `workspace` - 事件的项目空间标识符
* `topic` - 事件主题/类别（例如 "assets.new"）
* `source` - 事件来源（例如 "nuclei", "httpx"）
* `data_type` - 数据类型（例如 "subdomain", "url", "finding"）
* `data` - 事件负载（字符串或对象）

**返回：** `boolean` - 如果事件发送成功则返回 `true`

#### generate\_event\_from\_file(workspace, topic, source, data\_type, path)

为文件中每一非空行发出一个事件。

```javascript
generate_event_from_file("{{Workspace}}", "assets.new", "subfinder", "subdomain", "{{Output}}/subdomains.txt")
// 返回：42（生成的事件数）
```

**参数：**

* `workspace` - 事件的项目空间标识符
* `topic` - 事件主题/类别
* `source` - 事件来源
* `data_type` - 数据类型
* `path` - 包含数据的文件路径（每行一个条目）

**返回：** `integer` - 成功生成的事件计数

### Webhook 函数

向设置中配置的外部 Webhook 端点发送事件：

```yaml
- name: notify-finding
  type: function
  function: |
    notify_webhook("Critical vulnerability found on {{target}}")
```

#### notify\_webhook(message)

向所有已配置的 Webhook 发送纯文本消息。

```javascript
notify_webhook("Scan completed for example.com with 15 findings")
```

#### send\_webhook\_event(eventType, data)

向所有已配置的 Webhook 发送结构化事件。

```javascript
send_webhook_event("scan_complete", {
  target: "example.com",
  findings: 15,
  duration: "2h30m"
})
```

### 从外部系统（Webhook）

通过 API 接收 Webhook 以触发工作流：

```bash
curl -X POST http://localhost:8080/osm/api/events/emit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "topic": "external.webhook",
    "source": "github",
    "data_type": "push",
    "data": "{\"repository\": \"myorg/myapp\", \"branch\": \"main\"}"
  }'
```

## 事件触发器

### 触发器配置

在工作流 YAML 中配置事件触发器：

```yaml
name: probe-new-assets
kind: module

triggers:
  - name: on-subdomain-discovered
    on: event
    event:
      topic: "assets.new"
      filters:
        - "event.source == 'amass'"
        - "event.data_type == 'subdomain'"
      dedupe_key: "{{event.data.value}}"
      dedupe_window: "5m"
    input:
      type: event_data
      field: "value"
      name: target
    enabled: true

params:
  - name: target
    required: true

steps:
  - name: probe
    type: bash
    command: httpx -u {{target}} -o {{Output}}/httpx.json -json
```

### EventConfig 字段

| 字段               | 类型       | 描述                                                               |
| ------------------ | ---------- | ------------------------------------------------------------------ |
| `topic`            | string     | 要匹配的事件主题（支持通配符模式：`*`, `?`, `[`；空值表示匹配所有） |
| `filters`          | \[]string  | JavaScript 表达式（所有必须通过）                                   |
| `filter_functions` | \[]string  | 带有可用工具函数的 JavaScript 表达式（所有必须通过）                |
| `dedupe_key`       | string     | 去重键模板                                                         |
| `dedupe_window`    | string     | 去重时间窗口（例如 "5m"）                                          |

### TriggerInput 选项

触发器输入支持两种语法：旧语法和新的导出风格语法。

#### 新的导出风格语法（推荐）

使用简洁语法将多个事件字段映射到工作流变量：

```yaml
input:
  # 变量名: 表达式
  target: event_data.url
  asset_type: event_data.type
  source: event.source
  description: trim(event_data.desc)
```

**表达式类型：**

* `event_data.<field>` - 访问解析后的事件数据字段（例如 `event_data.url`, `event_data.severity`）
* `event.<field>` - 访问事件元数据（`event.topic`, `event.source`, `event.name`, `event.id`, `event.data_type`, `event.workspace`, `event.run_uuid`, `event.workflow_name`）
* `function(...)` - 使用工具函数转换值（例如 `trim(event_data.desc)`, `lower(event_data.name)`）

#### 旧语法

原始输入配置（仍支持向后兼容）：

| 类型         | 描述                     | 字段                                |
| ------------ | ------------------------ | ----------------------------------- |
| `event_data` | 从事件负载中提取         | `field` - 要提取的 JSON 路径        |
| `function`   | 使用函数转换             | `function` - JS 表达式（例如 jq）   |
| `param`      | 静态参数                 | `name` - 参数名称                   |
| `file`       | 从文件读取               | `path` - 文件路径                   |

```yaml
# 从事件数据中提取字段
input:
  type: event_data
  field: "url"
  name: target

# 使用 jq 函数转换
input:
  type: function
  function: 'jq("{{event.data}}", ".target.url")'
  name: target

# 使用静态参数
input:
  type: param
  name: default_target
```

### 主题通配符模式

事件主题支持通配符模式以实现灵活匹配：

```yaml
event:
  topic: "assets.*"         # 匹配 assets.new, assets.updated 等
  topic: "*.new"            # 匹配 assets.new, vulns.new 等
  topic: "scan.*.complete"  # 匹配 scan.nuclei.complete, scan.httpx.complete
  topic: "*"                # 匹配所有主题
```

| 模式     | 描述                     |
| -------- | ------------------------ |
| `*`      | 匹配任意字符序列         |
| `?`      | 匹配任意单个字符         |
| `[abc]`  | 匹配集合中的任意字符     |

### JavaScript 过滤器

过滤器是针对每个事件评估的 JavaScript 表达式。所有过滤器必须返回 `true` 才能触发工作流。

#### 可用事件字段

```javascript
event.topic      // "assets.new"
event.id         // "uuid-string"
event.name       // "subdomain.discovered"
event.source     // "amass"
event.data_type  // "subdomain"
event.data       // 解析后的 JSON 对象或原始字符串
event.workspace  // "myworkspace"
```

#### 过滤器示例

```yaml
filters:
  # 匹配特定来源
  - "event.source == 'nuclei'"

  # 从解析数据中匹配严重性
  - "event.data.severity == 'critical'"

  # 按数据类型匹配
  - "event.data_type == 'vulnerability'"

  # 组合条件（隐式 AND）
  - "event.source == 'nuclei'"
  - "event.data.severity == 'high' || event.data.severity == 'critical'"
```

#### 常见过滤器模式

```yaml
# 仅处理来自特定工具的事件
filters:
  - "event.source == 'subfinder' || event.source == 'amass'"

# 按严重性过滤
filters:
  - "['critical', 'high'].includes(event.data.severity)"

# 匹配 URL 模式
filters:
  - "event.data.url && event.data.url.includes('/api/')"

# 特定项目空间
filters:
  - "event.workspace == 'production'"
```

### 过滤器函数

对于更高级的过滤，使用 `filter_functions`，它提供对工具函数的访问，如 `contains()`, `starts_with()`, `ends_with()`, `file_exists()` 等：

```yaml
triggers:
  - name: on-api-endpoint
    on: event
    event:
      topic: "assets.new"
      # 简单过滤器（仅基础 JS）
      filters:
        - "event.source == 'httpx'"
      # 带有工具函数的过滤器函数
      filter_functions:
        - "contains(event.data.url, '/api/')"
        - "!ends_with(event.data.url, '.js')"
    input:
      type: event_data
      field: "url"
      name: target
    enabled: true
```

#### 可用过滤器函数

| 函数                       | 描述                     | 示例                                                  |
| -------------------------- | ------------------------ | ----------------------------------------------------- |
| `contains(str, substr)`    | 检查字符串是否包含子串   | `contains(event.data.url, '/api/')`                   |
| `starts_with(str, prefix)` | 检查字符串是否以前缀开头 | `starts_with(event.data.severity, 'critical')`        |
| `ends_with(str, suffix)`   | 检查字符串是否以后缀结尾 | `ends_with(event.data.url, '.json')`                  |
| `file_exists(path)`        | 检查文件是否存在         | `file_exists("{{event.data.output_path}}/results.json")` |
| `file_length(path)`        | 获取文件行数             | `file_length(event.data.results_file) > 0`            |
| `is_empty(str)`            | 检查字符串是否为空       | `!is_empty(event.data.target)`                        |
| `trim(str)`                | 去除首尾空白             | 在输入表达式中使用                                    |

#### 组合过滤器和过滤器函数

可以同时使用 `filters`（基础 JS）和 `filter_functions`（带工具函数）。两者中的所有表达式都必须通过：

```yaml
event:
  topic: "vulns.discovered"
  # 基础 JS 表达式
  filters:
    - "event.source == 'nuclei'"
  # 带工具函数的表达式
  filter_functions:
    - "contains(event.data.template_id, 'CVE')"
    - "starts_with(event.data.severity, 'critical') || starts_with(event.data.severity, 'high')"
```

### 事件信封模板变量

事件触发的工作流可以访问包含完整事件数据的特殊模板变量：

| 变量               | 描述                            |
| ------------------ | ------------------------------- |
| `{{EventEnvelope}}`  | 完整 JSON 编码的事件信封        |
| `{{EventTopic}}`     | 事件主题（例如 "assets.new"）   |
| `{{EventSource}}`    | 事件来源（例如 "nuclei", "httpx"） |
| `{{EventDataType}}`  | 数据类型（例如 "subdomain", "url"） |
| `{{EventTimestamp}}` | 事件时间戳（RFC3339 格式）      |
| `{{EventData}}`      | 原始事件数据负载                |

**使用示例：**

```yaml
name: event-processor
kind: module

triggers:
  - name: on-event
    on: event
    event:
      topic: "scan.complete"
    input:
      type: event_data
      field: "target"
      name: target
    enabled: true

steps:
  - name: log-event-details
    type: function
    functions:
      - 'log_info("Topic: {{EventTopic}}")'
      - 'log_info("Source: {{EventSource}}")'
      - 'log_info("Full envelope: {{EventEnvelope}}")'

  - name: save-envelope
    type: bash
    command: |
      echo '{{EventEnvelope}}' > {{Output}}/event-envelope.json
```

### 去重

使用时间窗口去重防止重复触发工作流：

```yaml
triggers:
  - name: on-unique-url
    on: event
    event:
      topic: "crawler.url"
      dedupe_key: "{{event.data.url}}"      # 唯一键模板
      dedupe_window: "5m"                    # 时间窗口
    input:
      type: event_data
      field: "value"
      name: target
```

去重键支持模板变量：

* `{{event.source}}-{{event.data.url}}` - 复合键
* `{{event.data.hash}}` - 简单字段

## 事件主题参考

### 内置主题

| 主题                | 描述                     | 典型来源     |
| ------------------- | ------------------------ | ------------ |
| `run.started`       | 工作流运行开始           | 引擎         |
| `run.completed`     | 工作流运行完成           | 引擎         |
| `run.failed`        | 工作流运行失败           | 引擎         |
| `step.completed`    | 单个步骤完成             | 引擎         |
| `step.failed`       | 单个步骤失败             | 引擎         |
| `asset.discovered`  | 发现新资产               | 侦察工具     |
| `asset.updated`     | 资产信息更新             | 扫描器       |
| `webhook.received`  | 收到外部 Webhook         | API          |
| `schedule.triggered`| 调度工作流触发           | 调度器       |

### 自定义主题

为领域特定事件定义自己的主题：

```javascript
// 自定义发现事件
generate_event("{{Workspace}}", "discovery.api-endpoint", "custom-scanner", "endpoint", {...})

// 自定义通知事件
generate_event("{{Workspace}}", "notification.slack", "workflow", "message", {...})
```

## 构建事件流水线

### 串联工作流

创建流水线，每个工作流触发下一个：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ 子域名枚举       │────▶│ HTTP 探测       │────▶│ 漏洞扫描        │
│                 │     │                 │     │                 │
│ 发出: assets.new│     │ 发出:           │     │ 发出:           │
│ (子域名)        │     │ assets.new      │     │ vulnerabilities │
│                 │     │ (存活主机)      │     │ .new            │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

#### 阶段 1：子域名枚举

```yaml
name: subdomain-enum
kind: module

steps:
  - name: enumerate
    type: bash
    command: |
      subfinder -d {{target}} -o {{Output}}/subdomains.txt
      amass enum -d {{target}} >> {{Output}}/subdomains.txt
      sort -u {{Output}}/subdomains.txt -o {{Output}}/subdomains.txt

  - name: emit-subdomains
    type: function
    function: |
      generate_event_from_fi
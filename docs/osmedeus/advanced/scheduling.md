> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 调度与队列

通过触发器自动化工作流执行。

## 触发器类型

| 类型     | 描述               | 用例                 |
| -------- | ------------------ | -------------------- |
| `cron`   | 基于时间的调度     | 每日/每周扫描        |
| `event`  | 事件驱动执行       | 响应新发现           |
| `watch`  | 文件变更检测       | 处理新数据           |
| `manual` | 仅按需触发         | 占位触发器           |

## CLI 快速入门

设置调度、队列和 Webhook 的最快方式——无需 YAML。

### 调度周期性扫描（`--as-cron`）

直接从 `run` 命令创建 cron 调度：

```bash
# 调度模块每天凌晨2点运行
osmedeus run -m subdomain-enum -t example.com --as-cron '0 2 * * *'

# 调度 Flow 每6小时运行一次
osmedeus run -f full-recon -t example.com --as-cron '0 */6 * * *'

# 带额外参数的调度
osmedeus run -m nuclei-scan -t example.com --as-cron '0 0 * * 1' -p threads=20
```

这会在数据库中创建一条调度记录。要激活它，请启动服务器：

```bash
osmedeus serve
```

列出已创建的调度：

```bash
osmedeus db ls --table schedules
```

### 将运行加入队列（`--queue`）

将运行加入队列以便延迟处理，而非立即执行：

```bash
# 为单个目标加入队列
osmedeus run -m subdomain-enum -t example.com --queue

# 为多个目标加入队列
osmedeus run -f full-recon -t example.com -t corp.com --queue

# 从目标文件加入队列
osmedeus run -m nuclei-scan -T targets.txt --queue
```

队列任务存储在数据库中，状态为 `queued`。使用以下命令处理它们：

```bash
# 处理队列任务（一次一个）
osmedeus worker queue run

# 带并发处理
osmedeus worker queue run --concurrency 5
```

服务器在运行时也会自动轮询队列任务（每30秒一次）。

### 注册 Webhook 触发器（`--as-webhook`）

创建一个按需触发运行的 Webhook URL：

```bash
# 注册一个 Webhook
osmedeus run -m subdomain-enum -t example.com --as-webhook

# 带身份验证密钥
osmedeus run -f full-recon -t example.com --as-webhook --webhook-auth-key mysecretkey
```

这会创建一个 Webhook 记录并打印触发 URL。使用以下命令触发它：

```bash
# GET 请求（简单触发）
curl http://localhost:8002/osm/api/webhook-runs/<uuid>/trigger

# 带身份验证密钥
curl http://localhost:8002/osm/api/webhook-runs/<uuid>/trigger?key=mysecretkey
```

需要在 `osm-settings.yaml` 中设置 `enable_trigger_via_webhook: true`，并且服务器正在运行。

## 队列管理

通过 `osmedeus worker queue` 子命令完全控制队列任务。

### 列出队列任务

```bash
# 列出所有队列任务
osmedeus worker queue list

# JSON 输出
osmedeus worker queue list --json
```

### 创建队列任务

`osmedeus run` 上 `--queue` 的替代方案：

```bash
# 将 Flow 加入队列
osmedeus worker queue new -f full-recon -t example.com

# 将模块加入队列，带多个目标
osmedeus worker queue new -m subdomain-enum -t example.com -t corp.com

# 从目标文件加入队列，带参数
osmedeus worker queue new -m nuclei-scan -T targets.txt -p threads=20
```

### 处理队列任务

```bash
# 开始处理（每5秒轮询数据库）
osmedeus worker queue run

# 带并发
osmedeus worker queue run --concurrency 5

# 带 Redis 连接实现双源轮询
osmedeus worker queue run --redis-url redis://localhost:6379
```

队列运行器每5秒轮询数据库。如果配置了 Redis，它还会通过 `BRPOP` 监听以降低延迟。

### 服务器端队列轮询

服务器（`osmedeus serve`）自动每30秒轮询队列任务。使用以下命令禁用：

```bash
osmedeus serve --no-queue-polling
```

## Webhook 管理

### 列出已注册的 Webhook

```bash
osmedeus worker webhooks
```

显示所有已注册 Webhook 触发器的表格，包含 UUID、工作流、目标、触发 URL 和身份验证密钥。

### 触发 Webhook

Webhook 可通过 GET 或 POST 触发：

```bash
# 简单 GET 触发
curl http://localhost:8002/osm/api/webhook-runs/<uuid>/trigger

# POST 带覆盖（目标、Flow 或模块）
curl -X POST http://localhost:8002/osm/api/webhook-runs/<uuid>/trigger \
  -H "Content-Type: application/json" \
  -d '{"target": "other.com"}'

# 覆盖工作流
curl -X POST http://localhost:8002/osm/api/webhook-runs/<uuid>/trigger \
  -H "Content-Type: application/json" \
  -d '{"target": "other.com", "module": "nuclei-scan"}'
```

POST 请求体字段（均为可选——默认值来自已注册的 Webhook）：

| 字段     | 描述                     |
| -------- | ------------------------ |
| `target` | 覆盖目标                 |
| `flow`   | 用 Flow 工作流覆盖       |
| `module` | 用模块工作流覆盖         |

### 身份验证

如果 Webhook 注册时使用了 `--webhook-auth-key`，则必须在查询参数中提供该密钥：

```bash
curl http://localhost:8002/osm/api/webhook-runs/<uuid>/trigger?key=mysecretkey
```

未提供正确密钥的请求将收到 `401 Unauthorized` 响应。

### Webhook 配置

必须在 `osm-settings.yaml` 中启用 Webhook 触发：

```yaml
server:
  enable_trigger_via_webhook: true
```

## 服务器标志

`osmedeus serve` 的相关标志：

| 标志                  | 描述                             |
| --------------------- | -------------------------------- |
| `--no-schedule`       | 禁用 cron/watch/event 调度器     |
| `--no-queue-polling`  | 禁用后台队列任务轮询             |
| `--no-event-receiver` | 禁用自动事件接收器               |
| `--no-hot-reload`     | 禁用配置热重载                   |

## Cron 触发器

使用 cron 表达式按计划执行工作流。

### 工作流定义

```yaml
kind: module
name: scheduled-scan

triggers:
  - name: daily-scan
    on: cron
    schedule: "0 2 * * *"    # 每天凌晨2点
    enabled: true

params:
  - name: target
    required: true

steps:
  - name: scan
    type: bash
    command: nuclei -u {{target}}
```

### Cron 表达式格式

```
┌───────────── 分钟 (0 - 59)
│ ┌───────────── 小时 (0 - 23)
│ │ ┌───────────── 日 (1 - 31)
│ │ │ ┌───────────── 月 (1 - 12)
│ │ │ │ ┌───────────── 星期 (0 - 6) (星期日 = 0)
│ │ │ │ │
* * * * *
```

### 常用调度

| 表达式           | 描述                 |
| ---------------- | -------------------- |
| `0 * * * *`      | 每小时               |
| `0 2 * * *`      | 每天凌晨2点          |
| `0 0 * * 0`      | 每周日               |
| `0 0 1 * *`      | 每月1号              |
| `*/15 * * * *`   | 每15分钟             |
| `0 9-17 * * 1-5` | 工作日9-17点每小时    |

## 事件触发器

响应事件执行工作流。

### 工作流定义

```yaml
kind: module
name: on-asset-discovered

triggers:
  - name: new-asset
    on: event
    event:
      topic: "assets.new"
      filters:
        - "event.source == 'subfinder'"
        - "event.data.type == 'subdomain'"
    input:
      type: event_data
      field: "url"
      name: target
    enabled: true

steps:
  - name: probe
    type: bash
    command: httpx -u {{target}}
```

### 事件主题

主题支持 glob 模式以实现灵活匹配：

```yaml
event:
  topic: "assets.*"         # 匹配 assets.new, assets.updated
  topic: "*.new"            # 匹配 assets.new, vulns.new
  topic: "*"                # 匹配所有主题
```

| 主题                 | 描述                 |
| --------------------- | -------------------- |
| `run.started`         | 工作流运行开始       |
| `run.completed`       | 工作流运行完成       |
| `run.failed`          | 工作流运行失败       |
| `step.completed`      | 步骤完成             |
| `assets.new`          | 发现新资产           |
| `vulnerabilities.new` | 发现新漏洞           |

### 事件过滤器

用于过滤事件的 JavaScript 表达式：

```yaml
filters:
  - "event.source == 'nuclei'"
  - "event.data.severity == 'critical'"
  - "event.workspace == 'example.com'"
```

如需使用工具函数进行高级过滤，请使用 `filter_functions`：

```yaml
event:
  topic: "assets.new"
  filters:
    - "event.source == 'httpx'"
  filter_functions:
    - "contains(event.data.url, '/api/')"
    - "!ends_with(event.data.url, '.js')"
```

可用过滤函数请参见 [事件驱动触发器](event-driven.mdx)。

### 事件输入映射

支持两种语法：

**新的 exports 风格语法（推荐）：**

```yaml
input:
  target: event_data.url
  source: event.source
  description: trim(event_data.desc)
```

**旧版语法：**

```yaml
input:
  type: event_data
  field: "target"      # 事件数据中的字段
  name: target         # 工作流参数名称
```

### 事件模板变量

事件触发的工作流可访问以下模板变量：

| 变量               | 描述                   |
| ------------------ | ---------------------- |
| `{{EventEnvelope}}`  | 完整 JSON 事件信封     |
| `{{EventTopic}}`     | 事件主题               |
| `{{EventSource}}`    | 事件来源               |
| `{{EventDataType}}`  | 数据类型               |
| `{{EventTimestamp}}` | 事件时间戳             |

## Watch 触发器

文件变更时执行工作流。使用 fsnotify 实现基于 inotify 的即时文件系统通知。

### 工作流定义

```yaml
kind: module
name: process-new-targets

triggers:
  - name: new-targets
    on: watch
    path: "/data/targets/"
    debounce: "500ms"    # 可选：防抖快速变化
    input:
      type: file_path
      name: target_file
    enabled: true

steps:
  - name: process
    type: foreach
    input: "{{target_file}}"
    variable: target
    step:
      type: bash
      command: scan {{target}}
```

### Watch 配置

| 字段       | 类型   | 描述                                           |
| ---------- | ------ | ---------------------------------------------- |
| `path`     | string | 要监视的目录或文件路径                         |
| `debounce` | string | 可选的防抖持续时间（例如 "500ms", "1s", "2s"） |

### 防抖

当文件被快速修改时（例如写入操作期间），可能会触发多个事件。使用 `debounce` 将快速变化合并为单个触发器：

```yaml
triggers:
  - name: watch-logs
    on: watch
    path: "/var/log/scan-results/"
    debounce: "1s"    # 最后一次变更后等待1秒再触发
    enabled: true
```

每次新文件事件都会重置防抖计时器。只有在指定时间内没有新事件时，工作流才会触发。

### Watch 事件

| 事件     | 描述         |
| -------- | ------------ |
| `create` | 创建新文件   |
| `modify` | 修改文件     |
| `delete` | 删除文件     |
| `rename` | 重命名文件   |

## API 管理

### 列出调度

```bash
curl http://localhost:8002/osm/api/schedules \
  -H "Authorization: Bearer $TOKEN"
```

### 创建调度

```bash
curl -X POST http://localhost:8002/osm/api/schedules \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "daily-scan",
    "workflow_name": "full-recon",
    "trigger_type": "cron",
    "schedule": "0 2 * * *",
    "input_config": {
      "target": "example.com"
    },
    "is_enabled": true
  }'
```

### 启用/禁用

```bash
# 启用
curl -X POST http://localhost:8002/osm/api/schedules/sched-123/enable \
  -H "Authorization: Bearer $TOKEN"

# 禁用
curl -X POST http://localhost:8002/osm/api/schedules/sched-123/disable \
  -H "Authorization: Bearer $TOKEN"
```

### 手动触发

```bash
curl -X POST http://localhost:8002/osm/api/schedules/sched-123/trigger \
  -H "Authorization: Bearer $TOKEN"
```

### 删除调度

```bash
curl -X DELETE http://localhost:8002/osm/api/schedules/sched-123 \
  -H "Authorization: Bearer $TOKEN"
```

### Webhook API 端点

列出已注册的 Webhook（需身份验证）：

```bash
curl http://localhost:8002/osm/api/webhook-runs/list \
  -H "Authorization: Bearer $TOKEN"
```

触发 Webhook（无需身份验证，除非设置了 `webhook_auth_key`）：

```bash
# GET
curl http://localhost:8002/osm/api/webhook-runs/<uuid>/trigger

# POST 带覆盖
curl -X POST http://localhost:8002/osm/api/webhook-runs/<uuid>/trigger \
  -H "Content-Type: application/json" \
  -d '{"target": "other.com", "module": "nuclei-scan"}'
```

## 组合触发器

一个工作流可以有多个触发器：

```yaml
kind: module
name: flexible-scan

triggers:
  # 每天运行
  - name: daily
    on: cron
    schedule: "0 2 * * *"
    enabled: true

  # 新资产时运行
  - name: on-asset
    on: event
    event:
      topic: "assets.new"
    input:
      type: event_data
      field: "url"
      name: target
    enabled: true

  # 手动触发占位
  - name: manual
    on: manual
    enabled: true

params:
  - name: target
    required: true

steps:
  - name: scan
    type: bash
    command: scan {{target}}
```

## 事件发射

从工作流中发射事件：

```yaml
- name: scan
  type: bash
  command: nuclei -u {{target}} -o {{Output}}/vulns.json
  on_success:
    - action: emit_event
      topic: "scan.completed"
      data:
        target: "{{target}}"
        results: "{{Output}}/vulns.json"
```

## 数据库存储

调度存储在数据库中：

```sql
-- schedules 表
id, name, workflow_name, workflow_path, trigger_type,
schedule, event_topic, watch_path, input_config,
is_enabled, last_run, next_run, run_count,
created_at, updated_at
```

查询调度：

```bash
osmedeus db query "SELECT * FROM schedules WHERE is_enabled = true"
```

## 最佳实践

1. **使用描述性触发器名称**
2. **测试时从禁用触发器开始**
3. **设置合理间隔**以避免过载
4. **使用事件过滤器**减少噪音
5. **通过事件日志监控触发器执行**
6. **结合分布式模式**实现扩展

## 故障排除

### 调度未运行

```bash
# 检查是否启用
curl http://localhost:8002/osm/api/schedules/sched-123

# 检查 next_run 时间
# 确保服务器正在运行
```

### 事件未触发

```bash
# 检查事件日志
curl "http://localhost:8002/osm/api/event-logs?topic=assets.new"

# 验证过滤器表达式
```

### Watch 未检测到变更

```bash
# 验证路径是否存在
# 检查文件权限
# 确保模式匹配
```

## 下一步

* [分布式执行](distributed.md) - 使用 Worker 扩展
* [服务器 CLI](../cli/server.md) - 服务器设置
* [API 概述](../api/overview.md) - 调度端点
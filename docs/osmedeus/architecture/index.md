> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 体系结构概述

> Osmedeus 工作流引擎的技术体系结构概述

# 体系结构概述

Osmedeus 是一个用于安全自动化的工作流引擎。它执行 YAML 定义的工作流，支持多种执行环境、分布式处理和广泛的自定义。

## 分层体系结构

```
+-------------------------------------------------------------+
|                     CLI / REST API                          |
|              (pkg/cli, pkg/server)                          |
+-------------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------------+
|                       执行器 (Executor)                      |
|              (internal/executor)                            |
|     协调工作流执行并管理状态                                 |
+-------------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------------+
|                    步骤调度器 (Step Dispatcher)              |
|                                                             |
|  +----------+ +----------+ +----------+ +----------+       |
|  |   Bash   | | Function | | Parallel | | Foreach  |       |
|  | 执行器   | | 执行器   | | 执行器   | | 执行器   |       |
|  +----------+ +----------+ +----------+ +----------+       |
|                                                             |
|  +----------+ +----------+ +----------+ +----------+       |
|  |  Remote  | |   HTTP   | |   LLM    | | Fragment |       |
|  |   Bash   | | 执行器   | | 执行器   | | 执行器   |       |
|  +----------+ +----------+ +----------+ +----------+       |
|                                                             |
+-------------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------------+
|                       Runner (Runner)                       |
|              (internal/runner)                              |
|                                                             |
|   +------------+  +------------+  +------------+           |
|   |    Host    |  |   Docker   |  |    SSH     |           |
|   |   Runner   |  |   Runner   |  |   Runner   |           |
|   +------------+  +------------+  +------------+           |
|                                                             |
+-------------------------------------------------------------+
```

## 核心包

| 包 (Package)          | 用途 (Purpose)                                                              |
| --------------------- | --------------------------------------------------------------------------- |
| `internal/core`       | 类型定义：Workflow, Step, Trigger, RunnerConfig, ExecutionContext           |
| `internal/parser`     | YAML 解析、验证和缓存 (Loader)                                              |
| `internal/executor`   | 工作流执行引擎，包含步骤调度                                                 |
| `internal/runner`     | 实现 Runner 接口的执行环境                                                   |
| `internal/template`   | `{{Variable}}` 插值引擎 (Engine, ShardedEngine)                             |
| `internal/functions`  | 通过 Goja JavaScript 运行时池提供的实用函数                                  |
| `internal/scheduler`  | Cron、事件和文件监视触发器                                                   |
| `internal/database`   | 通过 Bun ORM 使用 SQLite/PostgreSQL                                         |
| `internal/linter`     | 工作流验证和 lint 检查                                                      |
| `pkg/cli`             | Cobra CLI 命令                                                              |
| `pkg/server`          | Fiber REST API                                                              |
| `internal/snapshot`   | 项目空间导出/导入为 ZIP 归档                                                 |
| `internal/installer`  | 二进制安装（直接获取和 Nix）                                                 |
| `internal/state`      | 运行状态导出，用于调试                                                       |
| `internal/updater`    | 通过 GitHub releases 自更新                                                  |

## 工作流执行流程

```
+-------------------------------------------------------------+
| 1. CLI 解析参数                                             |
|    osmedeus run -f general -t example.com                   |
+-------------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------------+
| 2. 从 ~/osmedeus-base/osm-settings.yaml 加载设置             |
+-------------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------------+
| 3. 解析器加载 YAML 工作流                                   |
|    - 验证模式                                               |
|    - 解析包含（片段）                                       |
|    - 在 Loader 中缓存                                       |
+-------------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------------+
| 4. 执行器初始化上下文                                       |
|    - 注入内置变量（Target, Output 等）                      |
|    - 使用默认值加载参数                                     |
|    - 创建执行上下文                                         |
+-------------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------------+
| 5. 对于每个步骤：                                           |
|    +-----------------------------------------------------+  |
|    | a. 检查 depends_on（等待依赖）                      |  |
|    | b. 评估 pre_condition                               |  |
|    | c. 渲染模板                                         |  |
|    | d. 调度到适当的执行器                               |  |
|    | e. 通过 Runner 执行                                 |  |
|    | f. 捕获输出和导出                                   |  |
|    | g. 评估决策路由                                     |  |
|    | h. 处理 on_success/on_error                         |  |
|    +-----------------------------------------------------+  |
+-------------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------------+
| 6. 导出结果                                                 |
|    - 保存运行状态                                           |
|    - 更新数据库                                             |
|    - 生成产物                                               |
+-------------------------------------------------------------+
```

## 步骤类型路由

步骤调度器根据类型将步骤路由到适当的执行器：

```
+-------------------------------------------------------------+
|                    步骤调度器 (Step Dispatcher)              |
|                                                             |
|   步骤类型 (Step Type)          执行器 (Executor)           |
|   -------------------------------------------------         |
|   "bash"           ---------> BashExecutor --> Runner       |
|   "function"       ---------> FunctionExecutor --> Goja     |
|   "parallel-steps" ---------> ParallelExecutor              |
|   "foreach"        ---------> ForeachExecutor               |
|   "remote-bash"    ---------> RemoteBashExecutor --> SSH/Docker |
|   "http"           ---------> HTTPExecutor                  |
|   "llm"            ---------> LLMExecutor                   |
|   "fragment-step"  ---------> FragmentStepExecutor --> Inline |
|                                                             |
+-------------------------------------------------------------+
```

## 关键类型

### WorkflowKind

```go theme={null}
const (
    KindModule   WorkflowKind = "module"   // 单一单元工作流
    KindFlow     WorkflowKind = "flow"     // 编排模块
    KindFragment WorkflowKind = "fragment" // 可复用的步骤集合
)
```

### StepType

```go theme={null}
const (
    StepTypeBash         StepType = "bash"
    StepTypeFunction     StepType = "function"
    StepTypeParallel     StepType = "parallel-steps"
    StepTypeForeach      StepType = "foreach"
    StepTypeRemoteBash   StepType = "remote-bash"
    StepTypeHTTP         StepType = "http"
    StepTypeLLM          StepType = "llm"
    StepTypeFragmentStep StepType = "fragment-step"
)
```

### RunnerType

```go theme={null}
const (
    RunnerTypeHost   RunnerType = "host"   // 本地执行
    RunnerTypeDocker RunnerType = "docker" // Docker 容器
    RunnerTypeSSH    RunnerType = "ssh"    // 远程 SSH
)
```

### TriggerType

```go theme={null}
const (
    TriggerCron   TriggerType = "cron"   // Cron 调度
    TriggerEvent  TriggerType = "event"  // 事件驱动
    TriggerWatch  TriggerType = "watch"  // 文件系统监视
    TriggerManual TriggerType = "manual" // CLI 执行
)
```

## 步骤执行器

| 执行器 (Executor)         | 描述 (Description)            | Runner          |
| ------------------------- | ----------------------------- | --------------- |
| `BashExecutor`            | 执行 shell 命令               | Host/Docker/SSH |
| `FunctionExecutor`        | 执行实用函数                  | Goja runtime    |
| `ParallelExecutor`        | 并发步骤执行                  | Multiple        |
| `ForeachExecutor`         | 带并发的迭代                  | Multiple        |
| `RemoteBashExecutor`      | 远程命令执行                  | Docker/SSH      |
| `HTTPExecutor`            | HTTP 请求                     | Built-in        |
| `LLMExecutor`             | LLM API 调用                  | Built-in        |
| `FragmentStepExecutor`    | 内联片段执行                  | Dispatcher      |

## 模板引擎

模板引擎提供 `{{variable}}` 插值：

```
+-------------------------------------------------------------+
|                    模板引擎 (Template Engine)                |
|                                                             |
|   输入: "nuclei -l {{Output}}/urls.txt -o {{Output}}/out"   |
|                           |                                 |
|                           v                                 |
|   上下文: { Output: "/workspaces/example_com" }             |
|                           |                                 |
|                           v                                 |
|   输出: "nuclei -l /workspaces/example_com/urls.txt ..."    |
|                                                             |
|   特性:                                                     |
|   - 标准模板: {{variable}}                                  |
|   - 次要模板 (foreach): [[variable]]                        |
|   - 生成器: $rand(16), $uuid()                              |
|   - 分片缓存，支持高并发                                    |
|   - 工作流的预编译模板                                      |
|                                                             |
+-------------------------------------------------------------+
```

## 函数运行时

函数通过 Goja JavaScript 运行时池执行，并带有 VM 池：

```
+-------------------------------------------------------------+
|                    Goja 运行时池 (Goja Runtime Pool)         |
|                                                             |
|   +---------+  +---------+  +---------+  +---------+       |
|   |   VM1   |  |   VM2   |  |   VM3   |  |   VM4   |       |
|   | (空闲)  |  | (忙碌)  |  | (空闲)  |  | (忙碌)  |       |
|   +---------+  +---------+  +---------+  +---------+       |
|                                                             |
|   特性:                                                     |
|   - 池大小基于 CPU 核心数                                   |
|   - 无全局互斥锁，支持并行执行                              |
|   - 条件变量的惰性加载                                      |
|   - 每次执行的上下文隔离                                    |
|                                                             |
+-------------------------------------------------------------+
```

## 数据库模式

```
+-----------------+     +-----------------+
|   项目空间       |     |     资产        |
+-----------------+     +-----------------+
| id              |     | id              |
| name            |     | workspace       |--+
| target          |     | asset_value     |  |
| created_at      |     | asset_type      |  |
| total_subs      |     | url             |  |
| total_urls      |     | status_code     |  |
| ...             |     | created_at      |  |
+-----------------+     | updated_at      |  |
                        +-----------------+  |
+-----------------+                          |
| 漏洞             |                          |
+-----------------+                          |
| id              |                          |
| workspace       |--------------------------+
| asset_value     |
| vuln_info       |
| severity        |
| template_id     |
| created_at      |
+-----------------+
```

## 调度器

调度器管理自动化工作流触发器：

```
+-------------------------------------------------------------+
|                       调度器 (Scheduler)                     |
|                                                             |
|   触发器类型:                                                |
|   +------------------------------------------------------+  |
|   |  Cron        | 基于时间的调度（cron 语法）            |  |
|   |  Event       | 系统事件（assets.new 等）              |  |
|   |  Watch       | 文件系统变化（fsnotify）               |  |
|   |  Manual      | CLI 调用                              |  |
|   +------------------------------------------------------+  |
|                                                             |
|   事件主题:                                                  |
|   - assets.new         发现新资产                            |
|   - vulnerabilities.new 发现新漏洞                           |
|   - webhook.received   收到外部 webhook                     |
|   - db.change          数据库变更事件                       |
|   - watch.files        文件系统变更事件                     |
|                                                             |
+-------------------------------------------------------------+
```

## 决策路由

步骤支持使用 switch/case 语法的条件分支：

```yaml theme={null}
decision:
  switch: "{{variable}}"
  cases:
    "value1": { goto: step-a }
    "value2": { goto: step-b }
  default: { goto: fallback }
```

使用 `goto: _end` 终止工作流。

## 插件注册模式

步骤执行器注册在插件注册表中以实现可扩展性：

```go theme={null}
type StepExecutor interface {
    Name() string
    StepTypes() []core.StepType
    Execute(ctx context.Context, step *core.Step, execCtx *core.ExecutionContext) (*core.StepResult, error)
    CanHandle(stepType core.StepType) bool
}

// 注册
dispatcher.RegisterExecutor(NewBashExecutor())
dispatcher.RegisterExecutor(NewFunctionExecutor())
dispatcher.RegisterExecutor(NewFragmentStepExecutor())
// ...
```

## 设置

设置从 `~/osmedeus-base/osm-settings.yaml` 加载：

```yaml theme={null}
general:
  base_folder: ~/osmedeus-base
  workspaces_path: ~/workspaces-osmedeus
  binaries_path: ~/osmedeus-base/external-binaries

database:
  type: sqlite  # 或 postgres
  path: ~/osmedeus-base/osmedeus.db

notification:
  telegram:
    bot_token: ""
    chat_id: ""
  webhooks: []

cdn:
  enabled: false
  provider: s3
  bucket: ""
```

## 添加新功能

### 新步骤类型

1. 在 `internal/core/types.go` 中添加常量：
   ```go theme={null}
   StepTypeCustom StepType = "custom"
   ```

2. 在 `internal/executor/` 中创建实现 `StepExecutor` 的执行器：
   ```go theme={null}
   type CustomExecutor struct {}
   func (e *CustomExecutor) Name() string { return "custom" }
   func (e *CustomExecutor) StepTypes() []core.StepType { return []core.StepType{core.StepTypeCustom} }
   func (e *CustomExecutor) Execute(...) (*core.StepResult, error) { ... }
   ```

3. 在 `dispatcher.go` 中注册：
   ```go theme={null}
   dispatcher.RegisterExecutor(NewCustomExecutor())
   ```

### 新 Runner

1. 在 `internal/runner/` 中实现 Runner 接口：
   ```go theme={null}
   type CustomRunner struct {}
   func (r *CustomRunner) Execute(ctx context.Context, cmd string) (string, error) { ... }
   func (r *CustomRunner) Close() error { ... }
   ```

2. 添加类型常量并在 runner 工厂中注册。

### 新实用函数

1. 在 `internal/functions/` 中添加 Go 实现：
   ```go theme={null}
   func (vf *vmFunc) customFunc(call goja.FunctionCall) goja.Value { ... }
   ```

2. 在 `constants.go` 中添加常量：
   ```go theme={null}
   FnCustomFunc = "custom_func"
   ```

3. 在 `goja_runtime.go` 中注册：
   ```go theme={null}
   vm.Set(FnCustomFunc, vf.customFunc)
   ```

### 新 CLI 命令

1. 在 `pkg/cli/` 中创建：
   ```go theme={null}
   var customCmd = &cobra.Command{
       Use:   "custom",
       Short: "自定义命令",
       RunE:  func(cmd *cobra.Command, args []string) error { ... },
   }
   ```

2. 在 `init()` 中添加到 `rootCmd`。

### 新 API 端点

1. 在 `pkg/server/handlers/` 中添加处理函数：
   ```go theme={null}
   func CustomHandler(cfg *config.Config) fiber.Handler { ... }
   ```

2. 在 `s
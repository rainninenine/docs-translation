> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 开发

> 针对 Osmedeus 的开发与破解

本文档描述了 Osmedeus 的技术架构和开发实践。适用于希望理解、修改或扩展代码库的开发者。

## 目录

* [项目结构](#project-structure)
* [体系结构概述](#architecture-overview)
* [核心组件](#core-components)
* [工作流引擎](#workflow-engine)
* [执行管道](#execution-pipeline)
* [Runner 系统](#runner-system)
* [身份验证中间件](#authentication-middleware)
* [模板引擎](#template-engine)
* [函数注册表](#function-registry)
* [调度器系统](#scheduler-system)
* [工作流检查器](#workflow-linter)
* [数据库层](#database-layer)
* [测试](#testing)
* [添加新功能](#adding-new-features)
* [CLI 快捷键与技巧](#cli-shortcuts-and-tips)

## 项目结构

```
osmedeus/
├── cmd/osmedeus/           # 应用程序入口点
├── internal/               # 私有包
│   ├── client/             # 远程 API 客户端
│   ├── config/             # 设置管理
│   ├── console/            # 控制台输出捕获
│   ├── core/               # 核心类型（工作流、步骤、触发器 等）
│   ├── database/           # 通过 Bun ORM 的 SQLite/PostgreSQL
│   ├── distributed/        # 分布式执行（主/工作节点）
│   ├── executor/           # 工作流执行引擎
│   ├── functions/          # 工具函数（Goja JS 运行时）
│   ├── heuristics/         # 目标类型检测
│   ├── installer/          # 二进制安装（直接/Nix）
│   ├── linter/             # 工作流检查与验证
│   ├── logger/             # 结构化日志（Zap）
│   ├── parser/             # YAML 解析与缓存
│   ├── runner/             # 执行环境（主机/docker/ssh）
│   ├── scheduler/          # 触发器调度（cron/事件/监视）
│   ├── snapshot/           # 项目空间导出/导入
│   ├── state/              # 运行状态导出
│   ├── template/           # {{变量}} 插值引擎
│   ├── terminal/           # 终端 UI（颜色、表格、旋转器）
│   ├── updater/            # 通过 GitHub 发布的自更新
│   └── workspace/          # 项目空间管理
├── lib/                    # 共享库工具
├── pkg/                    # 公共包
│   ├── cli/                # Cobra CLI 命令
│   └── server/             # Fiber REST API 服务器
│       ├── handlers/       # 请求处理器
│       └── middleware/     # 身份验证中间件（JWT、API 密钥）
├── public/                 # 公共资源（示例、预设、UI）
├── test/                   # 测试套件
│   ├── e2e/                # E2E CLI 测试
│   ├── integration/        # 集成测试
│   └── testdata/           # 测试工作流夹具
├── docs/                   # API 文档
└── build/                  # 构建产物和 Docker 文件
```

## 体系结构概述

Osmedeus 采用分层架构：

```
┌─────────────────────────────────────────────────────────────┐
│                         CLI / API                            │
│              (pkg/cli, pkg/server)                          │
├─────────────────────────────────────────────────────────────┤
│                      执行器层                                │
│  ┌─────────────┐ ┌──────────────┐ ┌────────────────────┐   │
│  │  执行器     │ │  调度器      │ │  步骤执行器        │   │
│  │             │ │              │ │  (bash, function,  │   │
│  │             │ │              │ │   foreach, 等)     │   │
│  └─────────────┘ └──────────────┘ └────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                      Runner 层                               │
│  ┌──────────────┐ ┌───────────────┐ ┌─────────────────┐    │
│  │ 主机 Runner  │ │ Docker Runner │ │   SSH Runner    │    │
│  └──────────────┘ └───────────────┘ └─────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                     支持系统                                  │
│  ┌──────────────┐ ┌───────────────┐ ┌─────────────────┐    │
│  │   模板引擎   │ │   函数注册表   │ │   调度器        │    │
│  │              │ │               │ │   (触发器)      │    │
│  └──────────────┘ └───────────────┘ └─────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                      数据层                                  │
│  ┌──────────────┐ ┌───────────────┐ ┌─────────────────┐    │
│  │   解析器/    │ │   数据库      │ │   项目空间      │    │
│  │   加载器     │ │   (SQLite/PG) │ │   管理器        │    │
│  └──────────────┘ └───────────────┘ └─────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## 核心组件

### 工作流类型

```go
// internal/core/workflow.go

type Workflow struct {
    Kind         WorkflowKind  // "module" 或 "flow"
    Name         string
    Description  string
    Params       []Param
    Triggers     []Trigger
    Runner       RunnerType
    RunnerConfig *RunnerConfig
    Steps        []Step        // 对于模块
    Modules      []ModuleRef   // 对于 Flow
}
```

**模块**：具有顺序步骤的单一执行单元
**Flow**：通过依赖管理编排多个模块

### 步骤类型

```go
// internal/core/step.go

type Step struct {
    Name             string
    Type             StepType      // bash, function, foreach, parallel-steps, remote-bash, http, llm
    PreCondition     string        // 跳过条件
    Command          string        // 对于 bash/remote-bash
    Commands         []string      // 多个命令
    Function         string        // 对于 function 类型
    Input            string        // 对于 foreach
    Variable         string        // Foreach 变量名
    Threads          int           // Foreach 并行度
    Step             *Step         // 用于 foreach 的嵌套步骤
    ParallelSteps    []Step        // 用于 parallel-steps 类型
    StepRunner       RunnerType    // 对于 remote-bash：docker 或 ssh
    StepRunnerConfig *StepRunnerConfig // 用于 remote-bash 的 Runner 配置
    Exports          map[string]string
    OnSuccess        []Action
    OnError          []Action
    Decision         *DecisionConfig   // 条件分支（switch/case）
}
```

#### remote-bash 步骤类型

`remote-bash` 步骤类型允许按步骤执行 Docker 或 SSH，独立于模块级别的 Runner：

```yaml
steps:
  - name: docker-scan
    type: remote-bash
    step_runner: docker
    step_runner_config:
      image: alpine:latest
      volumes:
        - /data:/data
    command: nmap -sV {{target}}

  - name: ssh-scan
    type: remote-bash
    step_runner: ssh
    step_runner_config:
      host: "{{ssh_host}}"
      port: 22
      user: "{{ssh_user}}"
      key_file: ~/.ssh/id_rsa
    command: whoami && hostname
```

#### 决策路由（条件分支）

步骤可以包含决策路由，根据 switch/case 匹配跳转到不同步骤：

```yaml
steps:
  - name: detect-type
    type: bash
    command: echo "{{target_type}}"
    exports:
      detected_type: "output"
    decision:
      switch: "{{detected_type}}"
      cases:
        "domain":
          goto: subdomain-enum
        "ip":
          goto: port-scan
        "cidr":
          goto: network-scan
      default:
        goto: generic-recon

  - name: subdomain-enum
    type: bash
    command: subfinder -d {{target}}
    decision:
      switch: "always"
      cases:
        "always":
          goto: _end  # 特殊值，结束工作流
```

特殊值 `_end` 从当前步骤终止工作流执行。

### 执行上下文

```go
// internal/core/context.go

type ExecutionContext struct {
    WorkflowName string
    WorkflowKind WorkflowKind
    RunID        string
    Target       string
    Variables    map[string]interface{}
    Params       map[string]string
    Exports      map[string]interface{}
    StepIndex    int
    Logger       *zap.Logger
}
```

上下文在执行管道中传递并累积状态：

* 变量由执行器设置（内置变量）
* 参数由用户提供
* 导出是步骤输出，传播到后续步骤

## 工作流引擎

### 解析器

解析器（`internal/parser/parser.go`）处理 YAML 解析：

```go
type Parser struct{}

func (p *Parser) Parse(path string) (*core.Workflow, error)
func (p *Parser) Validate(workflow *core.Workflow) error
```

### 加载器

加载器（`internal/parser/loader.go`）提供缓存和查找：

```go
type Loader struct {
    workflowsDir string
    modulesDir   string
    cache        map[string]*core.Workflow
}

func (l *Loader) LoadWorkflow(name string) (*core.Workflow, error)
func (l *Loader) ListFlows() ([]string, error)
func (l *Loader) ListModules() ([]string, error)
```

查找顺序：

1. 检查缓存
2. 尝试 `workflows/<name>.yaml`
3. 尝试 `workflows/<name>-flow.yaml`
4. 尝试 `workflows/modules/<name>.yaml`
5. 尝试 `workflows/modules/<name>-module.yaml`

## 执行管道

### 流程

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   CLI/API    │────▶│   执行器     │────▶│   调度器     │
└──────────────┘     └──────────────┘     └──────────────┘
                                                 │
                    ┌────────────────────────────┼────────────────────────────┐
                    │                            │                            │
                    ▼                            ▼                            ▼
             ┌──────────────┐            ┌──────────────┐            ┌──────────────┐
             │ Bash执行器   │            │函数执行器    │            │Foreach执行器 │
             └──────────────┘            └──────────────┘            └──────────────┘
             ┌──────────────┐            ┌──────────────┐
             │ HTTP执行器   │            │ LLM执行器    │
             └──────────────┘            └──────────────┘
                    │                            │                            │
                    └────────────────────────────┼────────────────────────────┘
                                                 ▼
                                          ┌──────────────┐
                                          │    Runner    │
                                          └──────────────┘
```

### 执行器

```go
// internal/executor/executor.go

type Executor struct {
    templateEngine   *template.Engine
    functionRegistry *functions.Registry
    stepDispatcher   *StepDispatcher
}

func (e *Executor) ExecuteModule(ctx context.Context, module *core.Workflow,
                                  params map[string]string, cfg *config.Config) (*core.WorkflowResult, error)
func (e *Executor) ExecuteFlow(ctx context.Context, flow *core.Workflow,
                                params map[string]string, cfg *config.Config) (*core.WorkflowResult, error)
```

关键职责：

1. 使用内置变量初始化执行上下文
2. 创建并设置适当的 Runner
3. 遍历步骤，分派到适当的处理器
4. 处理前置条件、导出和决策路由
5. 处理 on_success/on_error 动作

### 步骤调度器

调度器使用插件注册表模式实现可扩展的步骤类型处理：

```go
// internal/executor/dispatcher.go

type StepDispatcher struct {
    registry         *PluginRegistry     // 可扩展的执行器注册表
    templateEngine   *template.Engine
    functionRegistry *functions.Registry
    bashExecutor     *BashExecutor       // 注册为插件
    llmExecutor      *LLMExecutor        // 注册为插件
    runner           runner.Runner
}

// PluginRegistry 管理步骤类型执行器
type PluginRegistry struct {
    executors map[core.StepType]StepExecutor
}

// StepExecutor 接口，用于所有步骤类型处理器
type StepExecutor interface {
    CanHandle(stepType core.StepType) bool
    Execute(ctx context.Context, step *core.Step, execCtx *core.ExecutionContext, runner runner.Runner) (*core.StepResult, error)
}

func (d *StepDispatcher) Dispatch(ctx context.Context, step *core.Step,
                                   execCtx *core.ExecutionContext) (*core.StepResult, error)
```

启动时注册的内置执行器：

* `BashExecutor` - 处理 `bash` 步骤
* `FunctionExecutor` - 处理 `function` 步骤
* `ForeachExecutor` - 处理 `foreach` 步骤
* `ParallelExecutor` - 处理 `parallel-steps` 步骤
* `RemoteBashExecutor` - 处理 `remote-bash` 步骤
* `HTTPExecutor` - 处理 `http` 步骤
* `LLMExecutor` - 处理 `llm` 步骤

## Runner 系统

### 接口

```go
// internal/runner/runner.go

type Runner interface {
    Execute(ctx context.Context, command string) (*CommandResult, error)
    Setup(ctx context.Context) error
    Cleanup(ctx context.Context) error
    Type() core.RunnerType
    IsRemote() bool
}

type CommandResult struct {
    Output   string
    ExitCode int
    Error    error
}
```

### 主机 Runner

使用 `os/exec` 的简单本地执行：

```go
func (r *HostRunner) Execute(ctx context.Context, command string) (*CommandResult, error) {
    cmd := exec.CommandContext(ctx, "sh", "-c", command)
    // ... 执行并捕获输出
}
```

### Docker Runner

支持临时（`docker run --rm`）和持久（`docker exec`）两种模式：

```go
type DockerRunner struct {
    config      *core.RunnerConfig
    containerID string  // 用于持久模式
}

func (r *DockerRunner) Execute(ctx context.Context, command string) (*CommandResult, error) {
    if r.config.Persistent && r.containerID != "" {
        return r.execInContainer(ctx, command)
    }
    return r.runEphemeral(ctx, command)
}
```

### SSH Runner

使用 `golang.org/x/crypto/ssh` 进行远程执行：

```go
type SSHRunner struct {
    config *core.RunnerConfig
    client *ssh.Client
}

func (r *SSHRunner) Setup(ctx context.Context) error {
    // 构建认证方法（密钥或密码）
    // 建立 SSH 连接
    // 可选地将二进制文件复制到远程
}
```

## 身份验证中间件

### 认证类型

服务器支持两种认证方法：

| 方法    | 标头                            | 描述                     |
| ------- | ------------------------------- | ------------------------ |
| API 密钥 | `x-osm-api-key`                 | 简单的基于令牌的认证     |
| JWT     | `Authorization: Bearer <token>` | 来自 `/osm/api/login` 的令牌 |

### 优先级逻辑

```go
// pkg/server/server.go - setupRoutes()

if s.config.Server.EnabledAuthAPI {
    api.Use(middleware.APIKeyAuth(s.config))
} else if !s.options.NoAuth {
    api.Use(middleware.JWTAuth(s.config))
}
```

优先级顺序：

1. **API 密钥认证** - 如果 `EnabledAuthAPI` 为 true
2. **JWT 认证** - 如果 API 密钥认证禁用且 NoAuth 为 false
3. **无认证** - 如果 NoAuth 选项为 true

### APIKeyAuth 实现

```go
// pkg/server/middleware/auth.go

func APIKeyAuth(cfg *config.Config) fiber.Handler {
    return func(c *fiber.Ctx) error {
        apiKey := c.Get("x-osm-api-key")
        if !isValidAPIKey(apiKey, cfg.Server.AuthAPIKey) {
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error":   true,
                "message": "无效或缺失 API 密钥",
            })
        }
        return c.Next()
    }
}
```

安全特性：

* 区分大小写的精确匹配
* 拒绝空/仅空白字符的密钥
* 拒绝占位符值（"null", "undefined", "nil"）

## 模板引擎

### 变量解析

模板引擎（`internal/template/engine.go`）处理 `{{variable}}` 插值：

```go
type Engine struct{}

func (e *Engine) Render(template string, ctx map[string]interface{}) (string, error)
```

解析顺序：

1. 检查上下文变量
2. 检查环境变量（可选）
3. 如果未找到，返回空字符串

### 内置变量注入

```go
// internal/executor/executor.go

func (e *Executor) injectBuiltinVariables(cfg *config.Config, params map[string]string,
                                           execCtx *core.ExecutionContext) {
    execCtx.SetVariable("BaseFolder", cfg.BaseFolder)
    execCtx.SetVariable("Target", params["target"])
    execCtx.SetVariable("Output", filepath.Join(workspacesPath, targetSpace))
    execCtx.SetVariable("threads", threads)
    execCtx.SetVariable("RunUUID", execCtx.RunUUID)
    // ... 更多变量
}
```

### Foreach 变量语法

Foreach 使用 `[[variable]]` 语法（双括号）以避免与模板变量冲突：

```yaml
- name: process-items
  type: foreach
  input: "/path/to/items.txt"
  variable: item
  step:
    command: echo [[item]]  # 在 foreach 迭代期间替换
```

## 函数注册表

### Otto JavaScript 运行时

函数在 Go 中实现，并暴露给 Otto JavaScript VM：

```go
// internal/functions/otto_runtime.go

type OttoRuntime struct {
    vm *otto.Otto
}

func NewOttoRuntime() *OttoRuntime {
    vm := otto.New()
    runtime := &OttoRuntime{vm: vm}
    runtime.registerFunctions()
    return runtime
}

func (r *OttoRuntime) registerFunctions() {
    r.vm.Set("fileExists", r.fileExists)
    r.vm.Set("fileLength", r.fileLength)
    r.vm.Set("trim", r.trim)
    // ... 注册所有函数
}
```

### 添加新函数

1. 在相应文件中添加 Go 实现：

```go
// internal/functions/file_functions.go

func (r *OttoRuntime) myNewFunction(call otto.FunctionCall) otto.Valu
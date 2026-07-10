> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 将 Osmedeus 用作库

> 在您的 Go 应用程序中嵌入 Osmedeus 工作流引擎

Osmedeus 可作为 Go 库使用，以便在您自己的应用程序中嵌入工作流执行能力。

## 安装

```bash theme={null}
go get github.com/j3ssie/osmedeus/v5
```

## 快速开始

```go theme={null}
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/j3ssie/osmedeus/v5/internal/config"
    "github.com/j3ssie/osmedeus/v5/internal/executor"
    "github.com/j3ssie/osmedeus/v5/internal/parser"
)

func main() {
    ctx := context.Background()

    // 1. 加载配置
    cfg, err := config.NewConfig("")
    if err != nil {
        log.Fatal(err)
    }

    // 2. 加载工作流
    loader := parser.NewLoader(cfg.WorkflowsPath)
    workflow, err := loader.LoadWorkflow("my-module")
    if err != nil {
        log.Fatal(err)
    }

    // 3. 创建并配置执行器
    exec := executor.NewExecutor()
    exec.SetVerbose(true)
    exec.SetLoader(loader)

    // 4. 执行工作流
    result, err := exec.ExecuteModule(ctx, workflow, map[string]string{
        "target": "example.com",
        "tactic": "default",
    }, cfg)

    if err != nil {
        log.Fatal(err)
    }

    // 5. 检查结果
    fmt.Printf("状态: %s\n", result.Status)
    for _, step := range result.Steps {
        fmt.Printf("  - %s: %s\n", step.StepName, step.Status)
    }
}
```

## 核心组件

### 配置

从默认位置（`~/osmedeus-base/osm-settings.yaml`）加载 Osmedeus 配置：

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/config"

// 从默认路径加载
cfg, err := config.NewConfig("")

// 从自定义路径加载
cfg, err := config.NewConfig("/path/to/osm-settings.yaml")

// 访问配置
fmt.Println("工作流路径:", cfg.WorkflowsPath)
fmt.Println("二进制文件路径:", cfg.BinariesPath)
fmt.Println("数据路径:", cfg.DataPath)
```

### 工作流解析器

解析并验证工作流 YAML 文件：

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/parser"

// 创建解析器
p := parser.NewParser()

// 从文件解析工作流
workflow, err := p.Parse("path/to/workflow.yaml")

// 从字节解析工作流
workflow, err := p.ParseContent([]byte(yamlContent))

// 验证工作流
if err := p.Validate(workflow); err != nil {
    log.Fatal("无效的工作流:", err)
}
```

### 工作流加载器

按名称加载工作流并支持缓存：

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/parser"

// 创建加载器
loader := parser.NewLoader("/path/to/workflows")

// 按名称加载（搜索 modules/ 和 flows/ 目录）
workflow, err := loader.LoadWorkflow("subdomain-enum")

// 按路径加载
workflow, err := loader.LoadWorkflow("/path/to/custom.yaml")

// 加载所有工作流
workflows, err := loader.LoadAllWorkflows()

// 从磁盘重新加载（清除缓存）
err := loader.ReloadWorkflows()

// 获取缓存的工作流
workflow, found := loader.GetWorkflow("name")
```

### 执行器

以编程方式执行工作流：

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/executor"

// 创建执行器
exec := executor.NewExecutor()

// 配置执行选项
exec.SetDryRun(true)              // 显示命令但不执行
exec.SetVerbose(true)             // 显示步骤输出
exec.SetSilent(true)              // 隐藏步骤输出
exec.SetSpinner(true)             // 显示旋转动画
exec.SetServerMode(true)          // 启用文件日志记录
exec.SetLoader(loader)            // 流程执行必需

// 执行模块工作流
result, err := exec.ExecuteModule(ctx, workflow, params, cfg)

// 执行流程工作流
result, err := exec.ExecuteFlow(ctx, flowWorkflow, params, cfg)
```

#### 执行参数

```go theme={null}
params := map[string]string{
    "target":           "example.com",   // 扫描目标
    "tactic":           "default",       // aggressive, default, gently
    "threads_hold":     "10",            // 覆盖线程数
    "workspace_prefix": "prefix",        // 项目空间文件夹后缀
    "workspaces_folder": "/path",        // 覆盖项目空间目录
    "exclude_modules":  "mod1,mod2",     // 要跳过的模块（流程）
    "heuristics_check": "basic",         // none, basic, advanced
}
```

#### 处理结果

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/core"

result, err := exec.ExecuteModule(ctx, workflow, params, cfg)
if err != nil {
    log.Fatal(err)
}

// 检查整体状态
switch result.Status {
case core.RunStatusCompleted:
    fmt.Println("工作流成功完成")
case core.RunStatusFailed:
    fmt.Println("工作流失败:", result.Error)
case core.RunStatusCancelled:
    fmt.Println("工作流已取消")
}

// 遍历步骤结果
for _, step := range result.Steps {
    fmt.Printf("步骤: %s, 状态: %s\n", step.StepName, step.Status)
    if step.Output != "" {
        fmt.Printf("  输出: %s\n", step.Output)
    }
}

// 访问导出
for key, value := range result.Exports {
    fmt.Printf("导出: %s = %v\n", key, value)
}
```

## 模板引擎

使用变量插值渲染模板字符串：

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/template"

// 创建引擎
engine := template.NewEngine()

// 渲染单个模板
result, err := engine.Render("Hello {{Name}}", map[string]any{
    "Name": "World",
})
// result = "Hello World"

// 渲染多个模板
results, err := engine.RenderMap(map[string]string{
    "greeting": "Hello {{Name}}",
    "command":  "scan {{Target}}",
}, variables)

// 渲染切片
results, err := engine.RenderSlice([]string{
    "{{var1}}", "{{var2}}",
}, variables)
```

### 次要变量（用于循环）

使用 `[[variable]]` 语法表示循环变量以避免冲突：

```go theme={null}
// 用于 foreach 上下文
result, err := engine.RenderSecondary(
    "Processing [[item]] with ID [[_id_]]",
    map[string]any{"item": "value", "_id_": 5},
)

// 检查模板是否使用次要分隔符
if engine.HasSecondaryVariable(template) {
    result, err = engine.RenderSecondary(template, vars)
}
```

### 生成器函数

在模板中执行生成器函数：

```go theme={null}
// 生成 UUID
uuid, err := engine.ExecuteGenerator("uuid()")

// 获取当前日期
date, err := engine.ExecuteGenerator("currentDate(\"2006-01-02\")")

// 获取环境变量
value, err := engine.ExecuteGenerator("getEnvVar(\"HOME\", \"/tmp\")")
```

## 函数注册表

以编程方式执行实用函数：

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/functions"

// 创建注册表
registry := functions.NewRegistry()

// 执行表达式
result, err := registry.Execute("trim('  hello  ')", variables)

// 评估条件（返回布尔值）
ok, err := registry.EvaluateCondition(
    "fileExists('{{Output}}/results.txt')",
    variables,
)

// 评估导出
exports, err := registry.EvaluateExports(map[string]string{
    "line_count": "fileLength('{{Output}}/data.txt')",
    "has_data":   "fileExists('{{Output}}/data.txt')",
}, variables)
```

### 可用函数

| 类别   | 函数                                                                         |
| ------ | ---------------------------------------------------------------------------- |
| 文件   | `fileExists`, `fileLength`, `readFile`, `writeFile`, `appendFile`, `createFolder` |
| 字符串 | `trim`, `split`, `contains`, `indexOf`, `toUpper`, `toLower`, `replace`           |
| JSON   | `jq`, `jqFromFile`, `jsonl2csv`, `filterJsonl`                                    |
| 数据库 | `db_select`, `db_insert`, `db_update`, `db_delete`                                |
| 日志   | `log_info`, `log_warn`, `log_error`                                               |

## 执行上下文

管理工作流执行状态：

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/core"

// 创建上下文
ctx := core.NewExecutionContext(
    "workflow-name",
    core.KindModule,
    "run-123",
    "example.com",
)

// 设置变量（线程安全）
ctx.SetVariable("key", "value")
ctx.SetParam("param_name", value)
ctx.SetExport("export_name", value)

// 获取变量
val, ok := ctx.GetVariable("key")
allVars := ctx.GetVariables()  // 用于模板渲染
```

## 数据库访问

连接到 Osmedeus 数据库：

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/database"

// 根据配置连接
db, err := database.Connect(cfg)

// 运行迁移
err = database.Migrate(context.Background())

// 使用 Bun ORM 查询
var workspaces []database.Workspace
err := db.NewSelect().
    Model(&workspaces).
    Where("name = ?", "example.com").
    Scan(context.Background())
```

### 数据库模型

```go theme={null}
// 项目空间
database.Workspace{
    Name        string
    Path        string
    TotalAssets int
}

// 资产
database.Asset{
    ID          string
    WorkspaceID string
    Type        string  // domain, subdomain, url
    Value       string
    Source      string
}

// 运行
database.Run{
    ID          string
    WorkspaceID string
    Status      string
    StartedAt   time.Time
    CompletedAt time.Time
}
```

## 核心类型

### 工作流类型

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/core"

// 工作流种类
core.KindModule   // 单一执行单元
core.KindFlow     // 编排模块
core.KindFragment // 可复用的步骤集合

// 步骤类型
core.StepTypeBash          // Shell 命令
core.StepTypeFunction      // 实用函数
core.StepTypeForeach       // 循环迭代
core.StepTypeParallelSteps // 并发执行
core.StepTypeRemoteBash    // 远程执行
core.StepTypeHTTP          // HTTP 请求
core.StepTypeLLM           // AI 驱动步骤

// Runner 类型
core.RunnerTypeHost   // 本地执行
core.RunnerTypeDocker // 容器执行
core.RunnerTypeSSH    // 远程 SSH 执行

// 运行状态
core.RunStatusPending
core.RunStatusRunning
core.RunStatusCompleted
core.RunStatusFailed
core.RunStatusCancelled
```

### 步骤结果

```go theme={null}
type StepResult struct {
    StepName  string
    Status    StepStatus
    Output    string
    Error     error
    Exports   map[string]interface{}
    Duration  time.Duration
    LogFile   string
}
```

### 工作流结果

```go theme={null}
type WorkflowResult struct {
    WorkflowName string
    WorkflowKind WorkflowKind
    RunID        string
    Target       string
    Status       RunStatus
    Steps        []*StepResult
    Exports      map[string]interface{}
    Error        error
}
```

## 创建自定义工作流

以编程方式构建工作流：

```go theme={null}
workflow := &core.Workflow{
    Kind:        core.KindModule,
    Name:        "custom-scan",
    Description: "自定义扫描工作流",
    Params: []core.Param{
        {Name: "target", Required: true},
    },
    Steps: []core.Step{
        {
            Name:    "enumerate",
            Type:    core.StepTypeBash,
            Command: "subfinder -d {{target}} -o {{Output}}/subs.txt",
            Exports: map[string]string{
                "sub_count": "fileLength('{{Output}}/subs.txt')",
            },
        },
        {
            Name:         "probe",
            Type:         core.StepTypeBash,
            Command:      "httpx -l {{Output}}/subs.txt -o {{Output}}/live.txt",
            PreCondition: "{{sub_count}} > 0",
        },
    },
}

// 执行前验证
p := parser.NewParser()
if err := p.Validate(workflow); err != nil {
    log.Fatal(err)
}

// 执行
result, err := exec.ExecuteModule(ctx, workflow, params, cfg)
```

## 工作流检查

使用检查器验证工作流：

```go theme={null}
import "github.com/j3ssie/osmedeus/v5/internal/linter"

// 创建检查器
l := linter.NewLinter()

// 检查工作流
issues := l.Lint(workflow)

for _, issue := range issues {
    fmt.Printf("[%s] %s: %s at %s\n",
        issue.Severity,  // info, warning, error
        issue.Rule,
        issue.Message,
        issue.Location,
    )
}

// 按严重级别过滤
warnings := l.LintWithSeverity(workflow, linter.SeverityWarning)
```

## 最佳实践

1. **始终在执行前验证工作流**
2. **对长时间运行的工作流使用上下文取消**
3. **适当处理每个步骤的错误**
4. **使用加载器进行流程执行（模块解析必需）**
5. **为网络操作设置适当的超时**
6. **使用 dry-run 模式测试工作流逻辑**

## 完整示例

```go theme={null}
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/j3ssie/osmedeus/v5/internal/config"
    "github.com/j3ssie/osmedeus/v5/internal/core"
    "github.com/j3ssie/osmedeus/v5/internal/database"
    "github.com/j3ssie/osmedeus/v5/internal/executor"
    "github.com/j3ssie/osmedeus/v5/internal/parser"
)

func main() {
    // 设置带取消的上下文
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 处理中断信号
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    go func() {
        <-sigCh
        fmt.Println("\n正在取消工作流...")
        cancel()
    }()

    // 加载配置
    cfg, err := config.NewConfig("")
    if err != nil {
        log.Fatal("加载配置失败:", err)
    }

    // 连接数据库
    _, err = database.Connect(cfg)
    if err != nil {
        log.Fatal("连接数据库失败:", err)
    }
    database.Migrate(ctx)

    // 创建工作流加载器
    loader := parser.NewLoader(cfg.WorkflowsPath)

    // 加载工作流
    workflow, err := loader.LoadWorkflow("subdomain-enum")
    if err != nil {
        log.Fatal("加载工作流失败:", err)
    }

    // 验证工作流
    p := parser.NewParser()
    if err := p.Validate(workflow); err != nil {
        log.Fatal("无效的工作流:", err)
    }

    // 创建并配置执行器
    exec := executor.NewExecutor()
    exec.SetVerbose(true)
    exec.SetLoader(loader)

    // 执行工作流
    result, err := exec.ExecuteModule(ctx, workflow, map[string]string{
        "target": "example.com",
        "tactic": "default",
    }, cfg)

    if err != nil {
        log.Fatal("执行失败:", err)
    }

    // 报告结果
    fmt.Printf("\n=== 工作流完成 ===\n")
    fmt.Printf("状态: %s\n", result.Status)
    fmt.Printf("执行的步骤数: %d\n", len(result.Steps))

    for _, step := range result.Steps {
        status := "✓"
        if step.Status != core.StepStatusSuccess {
            status = "✗"
        }
        fmt.Printf("  %s %s (%s)\n", status, step.StepName, step.Duration)
    }

    if len(result.Exports) > 0 {
        fmt.Println("\n导出:")
        for k, v := range result.Exports {
            fmt.Printf("  %s: %v\n", k, v)
        }
    }
}
```

## 下一步

* [扩展 Osmedeus](extending-osmedeus) - 添加自定义步骤类型和 Runner
* [开发](development) - 设置开发环境
* [REST API](../reference/rest-api) - 用于工作流执行的 HTTP API
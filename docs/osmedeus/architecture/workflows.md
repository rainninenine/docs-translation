> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 工作流体系结构

> 深入探讨 Osmedeus 工作流系统，包括片段和代码检查

# 工作流体系结构

工作流是 Osmedeus 中的核心抽象，通过 YAML 配置定义自动化安全任务。本文档涵盖工作流系统体系结构，包括片段和代码检查系统。

## 工作流类型

Osmedeus 支持三种工作流类型：

| 类型       | 用途                     | 包含内容       |
| ---------- | ------------------------ | -------------- |
| `module`   | 单一执行单元             | 步骤数组       |
| `flow`     | 编排模块                 | 模块数组       |
| `fragment` | 可复用的步骤集合         | 步骤数组       |

```
+-------------------------------------------------------------+
|                       Flow                                  |
|  +-------------+  +-------------+  +-------------+          |
|  |  Module A   |  |  Module B   |  |  Module C   |          |
|  |  +-------+  |  |  +-------+  |  |  +-------+  |          |
|  |  | Step  |  |  |  | Step  |  |  |  |Fragment|  |          |
|  |  | Step  |  |  |  | Step  |  |  |  | Step   |  |          |
|  |  |Fragment|  |  |  +-------+  |  |  | Step   |  |          |
|  |  +-------+  |  |              |  |  +-------+  |          |
|  +-------------+  +-------------+  +-------------+          |
+-------------------------------------------------------------+
```

## 工作流结构

### 工作流类型

```go
type Workflow struct {
    Kind         WorkflowKind  // module, flow, fragment
    Name         string        // 唯一标识符
    Description  string        // 人类可读的描述
    Tags         TagList       // 逗号分隔的标签
    Params       []Param       // 输入参数
    Triggers     []Trigger     // 自动触发
    Dependencies *Dependencies // 外部工具需求
    Reports      []Report      // 输出报告

    // 执行偏好
    Preferences  *Preferences  // 可选执行设置

    // Runner 配置（仅 module 类型）
    Runner       RunnerType    // host, docker, ssh
    RunnerConfig *RunnerConfig // Runner 特定配置

    // 模块特定字段
    Steps    []Step           // 执行步骤
    Includes []FragmentInclude // 片段包含

    // Flow 特定字段
    Modules  []ModuleRef      // 模块引用

    // 内部元数据
    FilePath string           // 源文件路径
    Checksum string           // 内容校验和
}
```

### 步骤类型

```go
type Step struct {
    Name         string      // 唯一步骤标识符
    Type         StepType    // 步骤类型
    DependsOn    []string    // 步骤依赖
    StepRunner   RunnerType  // 每步骤 Runner 覆盖
    PreCondition string      // 如果为 false 则跳过
    Log          string      // 日志文件路径
    Timeout      StepTimeout // 执行超时

    // Bash 步骤字段
    Command          string
    Commands         []string
    ParallelCommands []string
    StdFile          string    // Stdout 捕获文件

    // 结构化参数字段
    SpeedArgs  string
    ConfigArgs string
    InputArgs  string
    OutputArgs string

    // 函数步骤字段
    Function          string
    Functions         []string
    ParallelFunctions []string

    // 并行步骤字段
    ParallelSteps []Step

    // Foreach 步骤字段
    Input    string
    Variable string
    Threads  StepThreads
    Step     *Step

    // 远程 Bash 步骤字段
    StepRunnerConfig *StepRunnerConfig
    StepRemoteFile   string
    HostOutputFile   string

    // HTTP 步骤字段
    URL         string
    Method      string
    Headers     map[string]string
    RequestBody string

    // LLM 步骤字段
    Messages       []LLMMessage
    Tools          []LLMTool
    LLMConfig      *LLMStepConfig
    IsEmbedding    bool
    EmbeddingInput []string

    // 片段步骤字段
    FragmentName string            // 要执行的片段
    Override     map[string]string // 覆盖参数

    // 通用字段
    Exports   map[string]string
    OnSuccess []Action
    OnError   []Action
    Decision  *DecisionConfig
}
```

## 工作流继承

工作流通过 `extends` 字段支持继承，允许子工作流继承并覆盖父配置。

### 继承体系结构

```
+-------------------------------------------------------------+
|                  InheritanceResolver                        |
|                                                             |
|   子工作流                                                  |
|   +---------------------------------------------------------+
|   | extends: parent-workflow                                |
|   | override: { params: ..., steps: ... }                   |
|   +---------------------------------------------------------+
|                           |                                 |
|                           v                                 |
|   +---------------------------------------------------------+
|   | 1. 检查循环依赖                                        |
|   | 2. 加载父工作流                                        |
|   | 3. 递归解析父工作流的继承                              |
|   | 4. 验证类型兼容性                                      |
|   | 5. 合并父 -> 子，应用覆盖                              |
|   +---------------------------------------------------------+
|                           |                                 |
|                           v                                 |
|   合并后的工作流                                            |
|   +---------------------------------------------------------+
|   | 所有父字段 + 子覆盖                                    |
|   | ResolvedFrom: "parent-workflow"                         |
|   +---------------------------------------------------------+
+-------------------------------------------------------------+
```

### InheritanceResolver 类型

```go
type InheritanceResolver struct {
    loader    *Loader
    resolving map[string]bool  // 跟踪正在解析的工作流（循环检测）
    childPath string           // 当前子工作流的目录（用于相对路径解析）
}
```

### 解析过程

1. **循环检测**：跟踪正在解析的工作流以检测循环继承
2. **父工作流加载**：按名称（同一目录）或路径（相对/绝对）加载父工作流
3. **递归解析**：如果父工作流也扩展了其他工作流，则递归解析
4. **类型验证**：子工作流和父工作流必须具有匹配的 `kind`（module/flow）
5. **合并**：应用子工作流的直接字段和覆盖部分

### 覆盖模式

```go
const (
    OverrideModeReplace OverrideMode = "replace"   // 完全替换父项
    OverrideModePrepend OverrideMode = "prepend"   // 在父项之前添加子项
    OverrideModeAppend  OverrideMode = "append"    // 在父项之后添加子项（默认）
    OverrideModeMerge   OverrideMode = "merge"     // 按名称匹配，替换/移除/追加
)
```

### WorkflowOverride 类型

```go
type WorkflowOverride struct {
    Params       map[string]*ParamOverride  // 覆盖参数属性
    Steps        *StepsOverride             // 步骤覆盖（仅 module）
    Modules      *ModulesOverride           // 模块覆盖（仅 flow）
    Triggers     []Trigger                  // 完全替换触发器
    Dependencies *Dependencies              // 与父依赖合并
    Preferences  *Preferences               // 子覆盖父
    RunnerConfig *RunnerConfig              // 子覆盖父
    Runner       *RunnerType                // 覆盖 Runner 类型
}
```

### StepsOverride 类型

```go
type StepsOverride struct {
    Mode    OverrideMode  // replace, prepend, append, merge
    Steps   []Step        // 要添加/匹配的步骤
    Remove  []string      // 要移除的步骤名称（merge 模式）
    Replace []Step        // 按名称替换的步骤（merge 模式）
}
```

### 合并优先级

```
优先级：子直接字段 > 子覆盖 > 父

1. 从父工作流的克隆开始
2. 应用子直接字段（name, description, tags）
3. 按模式应用覆盖部分
4. 清除 extends 字段以防止重新解析
```

### 父工作流解析顺序

1. 子工作流同一目录（名称 + .yaml/.yml）
2. 从子工作流目录的相对路径
3. 按名称在工作流目录中搜索
4. 绝对路径

## 片段

片段是可复用的步骤集合，可以嵌入到模块中。

### 片段定义

```yaml
kind: fragment
name: notification-fragment
description: 通用通知步骤
params:
  - name: channel
    type: string
    default: "#security-alerts"
steps:
  - name: notify
    type: bash
    command: notify-send "{{channel}}" "{{message}}"
```

### 片段包含

模块可以使用 `includes` 字段包含片段：

```yaml
kind: module
name: subdomain-enum
includes:
  - name: notification-fragment
    as: notify
    params:
      channel: "#recon"
steps:
  - name: run-amass
    type: bash
    command: amass enum -d {{target}}
  - name: send-notification
    type: fragment
    fragment_name: notify
    override:
      message: "子域名枚举完成"
```

### FragmentInclude 类型

```go
type FragmentInclude struct {
    Name   string            // 片段工作流名称
    As     string            // 本地别名
    Params map[string]string // 参数覆盖
}
```

### 片段解析

1. **加载**：在工作流解析期间加载片段
2. **验证**：片段类型必须为 `fragment`
3. **参数绑定**：包含参数与片段默认值合并
4. **步骤展开**：片段步骤在执行时嵌入

### 片段步骤类型

```go
type Step struct {
    // ... 其他字段
    Type         StepType // "fragment"
    FragmentName string   // 来自 includes 的别名
    Override     map[string]string // 运行时参数覆盖
}
```

### 片段执行

当执行片段步骤时：

1. 通过别名从 includes 解析片段
2. 将覆盖参数与包含参数合并
3. 按顺序执行片段步骤
4. 返回合并结果

## 代码检查系统

工作流代码检查器验证 YAML 工作流的正确性和最佳实践。

### 代码检查体系结构

```
+-------------------------------------------------------------+
|                        Linter                               |
|                                                             |
|   YAML 源                                                  |
|   +---------------------------------------------------------+
|   | kind: module                                            |
|   | name: my-workflow                                       |
|   | steps: ...                                              |
|   +---------------------------------------------------------+
|                           |                                 |
|                           v                                 |
|  +---------------------------------------------------------+
|  |                   WorkflowAST                           |
|  |  - Workflow: *core.Workflow                             |
|  |  - Source: []byte                                       |
|  |  - Root: ast.Node                                       |
|  |  - NodeMap: map[string]ast.Node                         |
|  +---------------------------------------------------------+
|                           |                                 |
|                           v                                 |
|  +---------------------------------------------------------+
|  |                    Rules                                |
|  |  +------------------+  +------------------+              |
|  |  | MissingRequired  |  | DuplicateStepName|              |
|  |  +------------------+  +------------------+              |
|  |  +------------------+  +------------------+              |
|  |  |   EmptyStep      |  | UnusedVariable   |              |
|  |  +------------------+  +------------------+              |
|  |  +------------------+  +------------------+              |
|  |  |  InvalidGoto     |  |InvalidDependsOn  |              |
|  |  +------------------+  +------------------+              |
|  |  +------------------+  +------------------+              |
|  |  |CircularDependency|  |UndefinedVariable |              |
|  |  +------------------+  +------------------+              |
|  +---------------------------------------------------------+
|                           |                                 |
|                           v                                 |
|  +---------------------------------------------------------+
|  |                   LintResult                            |
|  |  - Issues: []LintIssue                                  |
|  |  - Errors: int                                          |
|  |  - Warnings: int                                        |
|  |  - Infos: int                                           |
|  +---------------------------------------------------------+
+-------------------------------------------------------------+
```

### 内置规则

| 规则                     | 严重级别 | 描述                                |
| ------------------------ | -------- | ------------------------------------------ |
| `missing-required-field` | warning  | 缺少必需字段（name, kind, type） |
| `duplicate-step-name`    | warning  | 多个步骤具有相同名称              |
| `empty-step`             | warning  | 步骤没有可执行内容             |
| `unused-variable`        | info     | 变量已导出但从未使用           |
| `undefined-variable`     | warning  | 变量被引用但未定义        |
| `invalid-goto`           | warning  | 决策 goto 引用了不存在的步骤 |
| `invalid-depends-on`     | warning  | depends_on 引用了不存在的步骤   |
| `circular-dependency`    | warning  | 检测到循环步骤依赖        |

### LinterRule 接口

```go
type LinterRule interface {
    Name() string                    // 唯一标识符
    Description() string             // 人类可读的描述
    Severity() Severity              // 默认严重级别
    Check(ast *WorkflowAST) []LintIssue
}
```

### LintIssue 类型

```go
type LintIssue struct {
    Rule       string   // 规则名称
    Severity   Severity // 问题严重级别
    Message    string   // 人类可读的描述
    Suggestion string   // 修复建议
    Line       int      // 基于 1 的行号
    Column     int      // 基于 1 的列号
    Field      string   // YAML 路径（例如 "steps[0].bash"）
}
```

### 运行代码检查器

#### CLI 用法

```bash
# 按名称验证
osmedeus workflow validate subdomain-enum

# 验证文件
osmedeus workflow lint ./my-workflow.yaml

# 验证文件夹
osmedeus workflow validate /path/to/workflows/

# CI 模式
osmedeus workflow lint . --check --format json
```

#### 输出格式

**Pretty（默认）**：

```
workflows/test.yaml:15:12 warning undefined-variable
  Variable 'unknown_var' is not defined
  Suggestion: Check that the variable is defined in params or a previous step's exports
```

**JSON**：

```json
{
  "file_path": "workflows/test.yaml",
  "issues": [
    {
      "rule": "undefined-variable",
      "severity": "warning",
      "message": "Variable 'unknown_var' is not defined",
      "line": 15,
      "column": 12,
      "field": "steps[2].command"
    }
  ]
}
```

**GitHub Actions**：

```
::warning file=workflows/test.yaml,line=15,col=12::undefined-variable: Variable 'unknown_var' is not defined
```

### 禁用规则

```bash
# 禁用特定规则
osmedeus workflow lint . --disable unused-variable,empty-step
```

### 自定义规则

实现 `LinterRule` 接口：

```go
type MyCustomRule struct{}

func (r *MyCustomRule) Name() string { return "my-custom-rule" }

func (r *MyCustomRule) Description() string {
    return "此规则检查内容的描述"
}

func (r *MyCustomRule) Severity() Severity { return SeverityWarning }

func (r *MyCustomRule) Check(wast *WorkflowAST) []LintIssue {
    var issues []LintIssue
    w := wast.Workflow

    // 实现你的验证逻辑
    for i, step := range w.Steps {
        if /* 条件 */ {
            line, col := wast.FindStepPosition(step.Name)
            issues = append(issues, LintIssue{
                Rule:       r.Name(),
                Severity:   r.Severity(),
                Message:    "问题描述",
                Suggestion: "如何修复",
                Line:       line,
                Column:     col,
                Field:      fmt.Sprintf("steps[%d]", i),
            })
        }
    }

    return issues
}

// 注册规则
linter := linter.NewDefaultLinter()
linter.RegisterRule(&MyCustomRule{})
```

## 决策路由

步骤支持条件分支：

```yaml
decision:
  switch: "{{status}}"
  cases:
    "critical": { goto: alert-step }
    "high": { goto: process-high }
    "none": { goto: _end }
  default: { goto: continue-step }
```

### DecisionConfig 类型

```go
type DecisionConfig struct {
    Switch  string                  // 要评估的变量
    Cases   map[string]DecisionCase // 情况映射
    Default *DecisionCase           // 默认情况
}

type DecisionCase struct {
    Goto string // 目标步骤名称或 "_end"
}
```

## 工作流执行上下文

```go
type ExecutionContext struct {
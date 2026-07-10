> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 添加步骤类型

添加自定义步骤执行器以扩展工作流能力。

## 概述

步骤类型是处理特定步骤配置的处理器。每种类型在 `internal/executor/` 中都有专用的执行器。

## 添加新步骤类型的步骤

### 1. 定义类型常量

添加到 `internal/core/types.go`：

```go theme={null}
// StepType 表示步骤的类型
type StepType string

const (
    StepTypeBash          StepType = "bash"
    StepTypeFunction      StepType = "function"
    StepTypeForeach       StepType = "foreach"
    StepTypeParallelSteps StepType = "parallel-steps"
    StepTypeRemoteBash    StepType = "remote-bash"
    StepTypeHTTP          StepType = "http"
    StepTypeLLM           StepType = "llm"
    StepTypeMyNew         StepType = "mynew"  // 添加你的新类型
)
```

### 2. 添加步骤字段（如果需要）

添加到 `internal/core/step.go`：

```go theme={null}
type Step struct {
    // 现有字段...

    // 新步骤类型的字段
    MyNewField     string            `yaml:"my_new_field,omitempty"`
    MyNewConfig    *MyNewConfig      `yaml:"my_new_config,omitempty"`
}

type MyNewConfig struct {
    Option1 string `yaml:"option1,omitempty"`
    Option2 int    `yaml:"option2,omitempty"`
}
```

### 3. 创建执行器

创建 `internal/executor/mynew_executor.go`：

```go theme={null}
package executor

import (
    "context"

    "github.com/osmedeus/osmedeus-ng/internal/core"
    "github.com/osmedeus/osmedeus-ng/internal/template"
)

type MyNewExecutor struct {
    templateEngine *template.Engine
}

func NewMyNewExecutor(templateEngine *template.Engine) *MyNewExecutor {
    return &MyNewExecutor{
        templateEngine: templateEngine,
    }
}

func (e *MyNewExecutor) Execute(
    ctx context.Context,
    step *core.Step,
    execCtx *core.ExecutionContext,
) (*core.StepResult, error) {
    // 1. 渲染步骤字段中的模板
    renderedField, err := e.templateEngine.Render(step.MyNewField, execCtx.Variables)
    if err != nil {
        return nil, fmt.Errorf("模板渲染失败: %w", err)
    }

    // 2. 执行你的步骤逻辑
    output, err := e.doSomething(ctx, renderedField, step.MyNewConfig)
    if err != nil {
        return &core.StepResult{
            Success: false,
            Error:   err,
        }, nil
    }

    // 3. 返回结果
    return &core.StepResult{
        Success: true,
        Output:  output,
    }, nil
}

func (e *MyNewExecutor) doSomething(
    ctx context.Context,
    field string,
    config *core.MyNewConfig,
) (string, error) {
    // 实现代码
    return "result", nil
}
```

### 4. 在调度器中注册

更新 `internal/executor/dispatcher.go`：

```go theme={null}
type StepDispatcher struct {
    bashExecutor       *BashExecutor
    functionExecutor   *FunctionExecutor
    foreachExecutor    *ForeachExecutor
    parallelExecutor   *ParallelExecutor
    remoteBashExecutor *RemoteBashExecutor
    httpExecutor       *HTTPExecutor
    llmExecutor        *LLMExecutor
    myNewExecutor      *MyNewExecutor  // 添加你的执行器
    runner             runner.Runner
}

func NewStepDispatcher(
    templateEngine *template.Engine,
    functionRegistry *functions.Registry,
    runner runner.Runner,
) *StepDispatcher {
    return &StepDispatcher{
        // 现有执行器...
        myNewExecutor: NewMyNewExecutor(templateEngine),
        runner:        runner,
    }
}

func (d *StepDispatcher) Dispatch(
    ctx context.Context,
    step *core.Step,
    execCtx *core.ExecutionContext,
) (*core.StepResult, error) {
    switch step.Type {
    case core.StepTypeBash:
        return d.bashExecutor.Execute(ctx, step, execCtx, d.runner)
    // ... 其他 case ...
    case core.StepTypeMyNew:
        return d.myNewExecutor.Execute(ctx, step, execCtx)
    default:
        return nil, fmt.Errorf("未知步骤类型: %s", step.Type)
    }
}
```

### 5. 添加验证

更新 `internal/parser/validator.go`：

```go theme={null}
func (v *Validator) validateStep(step *core.Step) error {
    switch step.Type {
    // ... 现有 case ...
    case core.StepTypeMyNew:
        return v.validateMyNewStep(step)
    }
    return nil
}

func (v *Validator) validateMyNewStep(step *core.Step) error {
    if step.MyNewField == "" {
        return fmt.Errorf("步骤 '%s': mynew 步骤类型需要 my_new_field", step.Name)
    }
    return nil
}
```

### 6. 编写测试

创建 `internal/executor/mynew_executor_test.go`：

```go theme={null}
package executor

import (
    "context"
    "testing"

    "github.com/osmedeus/osmedeus-ng/internal/core"
    "github.com/osmedeus/osmedeus-ng/internal/template"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestMyNewExecutor_Execute(t *testing.T) {
    engine := template.NewEngine()
    executor := NewMyNewExecutor(engine)

    step := &core.Step{
        Name:       "test-step",
        Type:       core.StepTypeMyNew,
        MyNewField: "test-value",
    }

    execCtx := &core.ExecutionContext{
        Variables: map[string]interface{}{
            "target": "example.com",
        },
    }

    result, err := executor.Execute(context.Background(), step, execCtx)

    require.NoError(t, err)
    assert.True(t, result.Success)
    assert.NotEmpty(t, result.Output)
}
```

## 示例：自定义通知步骤

```go theme={null}
// internal/executor/notify_executor.go

type NotifyExecutor struct {
    templateEngine *template.Engine
}

func (e *NotifyExecutor) Execute(
    ctx context.Context,
    step *core.Step,
    execCtx *core.ExecutionContext,
) (*core.StepResult, error) {
    // 渲染消息模板
    message, err := e.templateEngine.Render(step.NotifyMessage, execCtx.Variables)
    if err != nil {
        return nil, err
    }

    // 根据通道发送通知
    switch step.NotifyChannel {
    case "slack":
        err = e.sendSlack(ctx, step.NotifyConfig.WebhookURL, message)
    case "discord":
        err = e.sendDiscord(ctx, step.NotifyConfig.WebhookURL, message)
    case "telegram":
        err = e.sendTelegram(ctx, step.NotifyConfig, message)
    }

    if err != nil {
        return &core.StepResult{Success: false, Error: err}, nil
    }

    return &core.StepResult{Success: true, Output: "通知已发送"}, nil
}
```

在工作流中的使用：

```yaml theme={null}
- name: notify-complete
  type: notify
  notify_channel: slack
  notify_message: "扫描完成，目标: {{target}}"
  notify_config:
    webhook_url: "{{slack_webhook}}"
```

## 最佳实践

1. **始终渲染模板** 后再使用步骤字段
2. **支持上下文取消** 通过 `ctx.Done()`
3. **返回有意义的错误** 并附带上下文
4. **通过 StepResult 导出有用值**
5. **编写全面的测试**
6. **为必填字段添加验证规则**

## 下一步

* [添加 Runner](runners.md) - 自定义执行环境
* [添加函数](functions.md) - 工具函数
* [体系结构](../concepts/architecture.md) - 系统概述
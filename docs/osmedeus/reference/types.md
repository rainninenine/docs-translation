> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 类型参考

> 用于 Osmedeus 工作流的所有类型常量的完整参考

# 类型参考

本文档提供了 Osmedeus 工作流中使用的所有类型常量的全面参考。

## WorkflowKind

`WorkflowKind` 类型定义了工作流的类别。每个工作流必须声明恰好一种类型。

| 常量             | 值          | 描述                               |
| ---------------- | ----------- | ---------------------------------- |
| `KindModule`     | `"module"`  | 包含步骤的单一单元工作流           |
| `KindFlow`       | `"flow"`    | 编排多个模块                       |
| `KindFragment`   | `"fragment"`| 可复用的步骤集合，用于嵌入模块中   |

### 辅助方法

* `IsModule()` - 如果工作流是模块则返回 true
* `IsFlow()` - 如果工作流是 Flow 则返回 true
* `IsFragment()` - 如果工作流是片段则返回 true

## StepType

`StepType` 类型定义了模块或片段中步骤的执行行为。

| 常量                     | 值                    | 描述                               |
| ------------------------ | --------------------- | ---------------------------------- |
| `StepTypeBash`           | `"bash"`              | 本地执行 shell 命令                |
| `StepTypeFunction`       | `"function"`          | 执行工具函数                       |
| `StepTypeParallel`       | `"parallel-steps"`    | 并行执行步骤                       |
| `StepTypeForeach`        | `"foreach"`           | 遍历输入行                         |
| `StepTypeRemoteBash`     | `"remote-bash"`       | 远程执行命令（Docker/SSH）         |
| `StepTypeHTTP`           | `"http"`              | 发起 HTTP 请求                     |
| `StepTypeLLM`            | `"llm"`               | 与 LLM API 交互                    |
| `StepTypeFragmentStep`   | `"fragment-step"`     | 内联执行片段                       |

### 辅助方法

* `IsBashStep()` - 如果步骤是 bash 类型则返回 true
* `IsFunctionStep()` - 如果步骤是 function 类型则返回 true
* `IsParallelStep()` - 如果步骤是 parallel-steps 类型则返回 true
* `IsForeachStep()` - 如果步骤是 foreach 类型则返回 true
* `IsRemoteBashStep()` - 如果步骤是 remote-bash 类型则返回 true
* `IsHTTPStep()` - 如果步骤是 http 类型则返回 true
* `IsLLMStep()` - 如果步骤是 llm 类型则返回 true
* `IsFragmentStep()` - 如果步骤是 fragment-step 类型则返回 true

## TriggerType

`TriggerType` 类型定义了工作流的触发方式。

| 常量              | 值          | 描述                               |
| ----------------- | ----------- | ---------------------------------- |
| `TriggerCron`     | `"cron"`    | 通过 cron 表达式定时执行           |
| `TriggerEvent`    | `"event"`   | 由系统事件触发                     |
| `TriggerWatch`    | `"watch"`   | 由文件系统变化触发                 |
| `TriggerManual`   | `"manual"`  | 手动 CLI 执行                      |

## RunnerType

`RunnerType` 类型定义了步骤的执行环境。

| 常量                 | 值          | 描述                               |
| -------------------- | ----------- | ---------------------------------- |
| `RunnerTypeHost`     | `"host"`    | 在本地机器上执行（默认）           |
| `RunnerTypeDocker`   | `"docker"`  | 在 Docker 容器中执行               |
| `RunnerTypeSSH`      | `"ssh"`     | 通过 SSH 在远程机器上执行          |

## VariableType

`VariableType` 类型定义了工作流参数的输入验证。

| 常量               | 值             | 描述                   |
| ------------------ | -------------- | ---------------------- |
| `VarTypeDomain`    | `"domain"`     | 有效的域名             |
| `VarTypeSubdomain` | `"subdomain"`  | 有效的子域名           |
| `VarTypeURL`       | `"url"`        | 有效的 URL             |
| `VarTypeCIDR`      | `"cidr"`       | 有效的 CIDR 表示法     |
| `VarTypePath`      | `"path"`       | 文件系统路径           |
| `VarTypeFile`      | `"file"`       | 存在的文件路径         |
| `VarTypeFolder`    | `"folder"`     | 存在的文件夹路径       |
| `VarTypeNumber`    | `"number"`     | 数值                   |
| `VarTypeString`    | `"string"`     | 任意字符串值           |
| `VarTypeRepo`      | `"repo"`       | Git 仓库 URL           |

## TargetType

`TargetType` 类型定义了目标输入分类。

| 常量                    | 值             | 描述                   |
| ----------------------- | -------------- | ---------------------- |
| `TargetTypeDomain`      | `"domain"`     | 域名目标               |
| `TargetTypeSubdomain`   | `"subdomain"`  | 子域名目标             |
| `TargetTypeURL`         | `"url"`        | URL 目标               |
| `TargetTypeCIDR`        | `"cidr"`       | CIDR 范围目标          |
| `TargetTypeRepo`        | `"repo"`       | Git 仓库目标           |
| `TargetTypePath`        | `"path"`       | 文件系统路径目标       |
| `TargetTypeFile`        | `"file"`       | 文件目标               |
| `TargetTypeFolder`      | `"folder"`     | 文件夹目标             |
| `TargetTypeNumber`      | `"number"`     | 数值目标               |
| `TargetTypeString`      | `"string"`     | 字符串目标             |

## ActionType

`ActionType` 类型定义了 `on_success` 和 `on_error` 事件的处理程序。

| 常量               | 值            | 描述                   |
| ------------------ | ------------- | ---------------------- |
| `ActionLog`        | `"log"`       | 记录消息               |
| `ActionAbort`      | `"abort"`     | 中止工作流执行         |
| `ActionContinue`   | `"continue"`  | 即使出错也继续执行     |
| `ActionExport`     | `"export"`    | 导出变量               |
| `ActionRun`        | `"run"`       | 运行命令或函数         |
| `ActionNotify`     | `"notify"`    | 发送通知               |

## StepStatus

`StepStatus` 类型表示步骤的执行状态。

| 常量                  | 值           | 描述                   |
| --------------------- | ------------ | ---------------------- |
| `StepStatusPending`   | `"pending"`  | 步骤等待执行           |
| `StepStatusRunning`   | `"running"`  | 步骤正在执行           |
| `StepStatusSuccess`   | `"success"`  | 步骤成功完成           |
| `StepStatusFailed`    | `"failed"`   | 步骤失败               |
| `StepStatusSkipped`   | `"skipped"`  | 步骤被跳过             |

## RunStatus

`RunStatus` 类型表示整体运行状态。

| 常量                   | 值             | 描述                   |
| ---------------------- | -------------- | ---------------------- |
| `RunStatusPending`     | `"pending"`    | 运行等待开始           |
| `RunStatusRunning`     | `"running"`    | 运行进行中             |
| `RunStatusCompleted`   | `"completed"`  | 运行成功完成           |
| `RunStatusFailed`      | `"failed"`     | 运行失败               |
| `RunStatusCancelled`   | `"cancelled"`  | 运行被取消             |
| `RunStatusSkipped`     | `"skipped"`    | 运行被跳过             |

## Severity（Linter）

`Severity` 类型定义了 lint 问题的严重级别。

| 常量              | 值    | 描述               |
| ----------------- | ----- | ------------------ |
| `SeverityInfo`    | `0`   | 信息性问题         |
| `SeverityWarning` | `1`   | 警告性问题         |
| `SeverityError`   | `2`   | 错误性问题         |

## OutputFormat（Linter）

`OutputFormat` 类型定义了 linter 的输出格式。

| 常量             | 值          | 描述                               |
| ---------------- | ----------- | ---------------------------------- |
| `FormatPretty`   | `"pretty"`  | 带上下文的彩色终端输出             |
| `FormatJSON`     | `"json"`    | 机器可读的 JSON                   |
| `FormatGitHub`   | `"github"`  | GitHub Actions 注释                |

## OverrideMode（继承）

`OverrideMode` 类型定义了工作流继承的合并策略。

| 常量                    | 值           | 描述                                               |
| ----------------------- | ------------ | -------------------------------------------------- |
| `OverrideModeReplace`   | `"replace"`  | 用子项完全替换父项                                 |
| `OverrideModePrepend`   | `"prepend"`  | 将子项添加到父项之前                               |
| `OverrideModeAppend`    | `"append"`   | 将子项添加到父项之后（默认）                       |
| `OverrideModeMerge`     | `"merge"`    | 按名称匹配：替换、追加新项、移除指定项             |

用于继承工作流时的 `override.steps.mode` 和 `override.modules.mode` 字段。

## 汇总表

| 类别           | 类型             | 值                                                                                             |
| -------------- | ---------------- | ---------------------------------------------------------------------------------------------- |
| 工作流         | `WorkflowKind`   | `module`, `flow`, `fragment`                                                                   |
| 步骤           | `StepType`       | `bash`, `function`, `parallel-steps`, `foreach`, `remote-bash`, `http`, `llm`, `fragment-step` |
| 触发           | `TriggerType`    | `cron`, `event`, `watch`, `manual`                                                             |
| Runner         | `RunnerType`     | `host`, `docker`, `ssh`                                                                        |
| 变量           | `VariableType`   | `domain`, `subdomain`, `url`, `cidr`, `path`, `file`, `folder`, `number`, `string`, `repo`     |
| 目标           | `TargetType`     | `domain`, `subdomain`, `url`, `cidr`, `repo`, `path`, `file`, `folder`, `number`, `string`     |
| 动作           | `ActionType`     | `log`, `abort`, `continue`, `export`, `run`, `notify`                                          |
| 状态           | `StepStatus`     | `pending`, `running`, `success`, `failed`, `skipped`                                           |
| 运行           | `RunStatus`      | `pending`, `running`, `completed`, `failed`, `cancelled`, `skipped`                            |
| Lint 严重级别  | `Severity`       | `info`, `warning`, `error`                                                                     |
| Lint 格式      | `OutputFormat`   | `pretty`, `json`, `github`                                                                     |
| 覆盖模式       | `OverrideMode`   | `replace`, `prepend`, `append`, `merge`                                                        |
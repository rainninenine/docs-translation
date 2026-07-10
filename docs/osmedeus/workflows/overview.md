> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 工作流概述

> 工作流结构与概念介绍

工作流是定义自动化扫描管线的 YAML 文件。

## 基本结构

### 模块

```yaml theme={null}
kind: module                    # 类型：module 或 flow
name: my-workflow               # 唯一工作流名称
description: What it does       # 人类可读的描述
tags:                           # 可选分类
  - reconnaissance
  - subdomain

params:                         # 输入参数
  - name: target
    required: true

runner: host                    # 执行环境
runner_config: {}               # Runner 选项

triggers:                        # 调度选项
  - name: daily
    on: cron
    schedule: "0 2 * * *"

steps:                          # 执行步骤
  - name: step-one
    type: bash
    command: echo "Hello"
```

### Flow

```yaml theme={null}
kind: flow
name: my-pipeline
description: Multi-module pipeline

params:
  - name: target
    required: true

modules:                        # 模块引用
  - name: recon
    path: modules/recon.yaml
  - name: scan
    path: modules/scan.yaml
    depends_on: [recon]
```

## 字段参考

### 顶层字段

| 字段             | 必需   | 描述                                       |
| ---------------- | ------ | ------------------------------------------ |
| `kind`           | 是     | `module` 或 `flow`                         |
| `name`           | 是     | 唯一工作流标识符                           |
| `description`    | 否     | 人类可读的描述                             |
| `tags`           | 否     | 分类标签数组                               |
| `params`         | 否     | 输入参数定义                               |
| `runner`         | 否     | 默认 Runner 类型（`host`、`docker`、`ssh`）|
| `runner_config`  | 否     | Runner 配置对象                            |
| `trigger`        | 否     | 调度触发器定义                             |
| `steps`          | 模块   | 执行步骤列表                               |
| `modules`        | Flow   | 模块引用列表                               |

### 参数

```yaml theme={null}
params:
  - name: target              # 参数名称（必需）
    required: true            # 必须提供（默认：false）
    default: ""               # 默认值
    description: Target domain
```

在模板中使用 `{{target}}`。

### 标签

```yaml theme={null}
tags:
  - reconnaissance
  - subdomain
  - passive
```

过滤工作流：

```bash theme={null}
osmedeus workflow list --tags reconnaissance
```

## 工作流类型

### 模块工作流

用于单一、聚焦的任务：

```yaml theme={null}
kind: module
name: subdomain-enum

params:
  - name: target
    required: true

steps:
  - name: subfinder
    type: bash
    command: subfinder -d {{target}} -o {{Output}}/subs.txt

  - name: amass
    type: bash
    command: amass enum -passive -d {{target}} >> {{Output}}/subs.txt

  - name: dedupe
    type: bash
    command: sort -u {{Output}}/subs.txt -o {{Output}}/subdomains.txt
```

### Flow 工作流

用于多阶段管线：

```yaml theme={null}
kind: flow
name: full-recon

params:
  - name: target
    required: true

modules:
  - name: subdomain-enum
    path: modules/subdomain-enum.yaml

  - name: http-probe
    path: modules/http-probe.yaml
    depends_on: [subdomain-enum]

  - name: screenshot
    path: modules/screenshot.yaml
    depends_on: [http-probe]
    condition: 'fileLength("{{Output}}/live.txt") > 0'
```

## 模块引用（Flow 中）

```yaml theme={null}
modules:
  - name: module-name          # 引用名称
    path: modules/file.yaml    # 模块 YAML 路径
    depends_on: [dep1, dep2]   # 等待这些模块完成
    condition: 'expression'    # 条件为 false 时跳过
    params:                    # 覆盖参数
      key: value
```

## 工作流存放位置

将工作流存储在工作流文件夹中（默认：`~/osmedeus-base/workflows/`）：

```
workflows/
├── modules/
│   ├── subdomain-enum.yaml
│   ├── http-probe.yaml
│   └── nuclei-scan.yaml
└── flows/
    ├── basic-recon.yaml
    └── full-assessment.yaml
```

## 运行工作流

```bash theme={null}
# 运行模块
osmedeus run -m subdomain-enum -t example.com

# 运行 Flow
osmedeus run -f full-recon -t example.com

# 列出可用工作流
osmedeus workflow list

# 显示工作流详情
osmedeus workflow show subdomain-enum

# 验证工作流
osmedeus workflow validate subdomain-enum
```

## 验证与 Linting

### 解析器验证

解析器验证以下内容：

1. **必需字段**：`kind`、`name`、`steps`（模块）或 `modules`（Flow）
2. **有效类型**：必须为 `module` 或 `flow`
3. **步骤名称**：每个步骤必须有唯一名称
4. **步骤类型**：必须有效（bash、function、foreach、parallel-steps、remote-bash、http、llm、agent）
5. **模块路径**：引用的模块必须存在（Flow）
6. **循环依赖**：依赖图中无循环（Flow）

### 工作流 Linter

工作流 Linter 在基本解析之外提供额外的最佳实践检查。它有助于在运行时之前发现潜在问题，同时允许工作流即使有警告也能执行。

```bash theme={null}
# Lint 工作流
osmedeus workflow lint my-workflow.yaml
osmedeus workflow lint my-workflow              # 按名称
osmedeus workflow lint /path/to/workflows/      # 目录

# 输出格式
osmedeus workflow lint workflow.yaml --format pretty   # 默认
osmedeus workflow lint workflow.yaml --format json     # 机器可读
osmedeus workflow lint workflow.yaml --format github   # CI 注释

# 按严重级别过滤
osmedeus workflow lint workflow.yaml --severity info     # 所有问题（默认）
osmedeus workflow lint workflow.yaml --severity warning  # 警告和错误
osmedeus workflow lint workflow.yaml --severity error    # 仅错误

# 禁用特定规则
osmedeus workflow lint workflow.yaml --disable unused-variable

# CI 模式（错误时退出码为 1）
osmedeus workflow lint workflow.yaml --check
```

### Linter 规则参考

| 规则                       | 严重级别 | 描述                           |
| -------------------------- | -------- | ------------------------------ |
| `missing-required-field`   | warning  | 缺少 name、kind 或 type 字段   |
| `duplicate-step-name`      | warning  | 多个步骤同名                   |
| `empty-step`               | warning  | 步骤无可执行内容               |
| `unused-variable`          | info     | 导出的变量从未被引用           |
| `invalid-goto`             | warning  | Decision goto 指向不存在的步骤 |
| `invalid-depends-on`       | warning  | 依赖不存在的步骤               |
| `circular-dependency`      | warning  | 步骤依赖形成循环               |

<Note>
  `undefined-variable` 规则存在但默认未启用，因为内置变量数量庞大。工作流可以带着 linter 警告执行——linter 旨在帮助识别潜在问题，而非阻止执行。
</Note>

### 严重级别

* **info** - 最佳实践建议（例如未使用的导出）
* **warning** - 可能导致问题的潜在问题（例如重复名称）
* **error** - 很可能导致失败的关键问题

## 工作流继承

工作流可以使用 `extends` 字段扩展父工作流。这允许：

* 在工作流间复用通用配置
* 创建专用变体（例如快速、激进、隐蔽）
* 保持一致的基工作流，同时进行针对性覆盖

### 基本继承

```yaml theme={null}
kind: module
name: subdomain-enum-fast
extends: subdomain-enum-base

# 覆盖描述
description: "Fast subdomain enumeration (reduced timeout)"

# override 部分指定要更改的内容
override:
  params:
    threads:
      default: 50
    timeout:
      default: "10m"
```

### Extends 字段

`extends` 字段指定要继承的父工作流：

```yaml theme={null}
extends: parent-workflow-name     # 按名称（同一目录）
extends: ./parent.yaml            # 按相对路径
extends: modules/base-module.yaml # 按工作流目录下的路径
```

子工作流继承父工作流的所有字段，子工作流的字段优先级更高。

### 覆盖模式

覆盖步骤（模块）或模块（Flow）时，可以指定合并策略：

| 模式      | 描述                                                   |
| --------- | ------------------------------------------------------ |
| `replace` | 完全用子项替换父项                                     |
| `prepend` | 将子项添加到父项之前                                   |
| `append`  | 将子项添加到父项之后（默认）                           |
| `merge`   | 按名称匹配：替换匹配项，追加新项，移除指定项           |

#### Append 模式（默认）

```yaml theme={null}
kind: module
name: extended-enum
extends: base-enum

override:
  steps:
    mode: append
    steps:
      - name: additional-step
        type: bash
        command: "extra-tool -t {{Target}}"
```

#### Prepend 模式

```yaml theme={null}
override:
  steps:
    mode: prepend
    steps:
      - name: setup-step
        type: bash
        command: "setup-tool"
```

#### Replace 模式

```yaml theme={null}
override:
  steps:
    mode: replace
    steps:
      - name: only-step
        type: bash
        command: "completely-different-tool"
```

#### Merge 模式

```yaml theme={null}
override:
  steps:
    mode: merge
    # 按名称替换现有步骤
    replace:
      - name: subfinder
        type: bash
        command: "subfinder -d {{Target}} --all -o {{Output}}/subfinder.txt"
    # 按名称移除步骤
    remove:
      - amass-passive
    # 追加新步骤
    steps:
      - name: new-tool
        type: bash
        command: "new-tool -t {{Target}}"
```

### 覆盖部分

`override` 块支持以下部分：

| 部分             | 描述                                           |
| ---------------- | ---------------------------------------------- |
| `params`         | 覆盖参数默认值、类型或必需状态                 |
| `steps`          | 覆盖步骤（仅模块工作流）                       |
| `modules`        | 覆盖模块（仅 Flow 工作流）                     |
| `triggers`       | 完全替换父触发器                               |
| `dependencies`   | 与父依赖合并                                   |
| `preferences`    | 覆盖执行偏好                                   |
| `runner_config`  | 覆盖 Runner 配置                               |
| `runner`         | 覆盖 Runner 类型                               |

### 参数覆盖

覆盖特定参数属性：

```yaml theme={null}
override:
  params:
    threads:
      default: 50           # 覆盖默认值
    wordlist:
      default: "{{Data}}/fast-wordlist.txt"
    new_param:              # 添加新参数
      default: "value"
      required: false
```

### 多层继承

工作流可以形成继承链：

```yaml theme={null}
# base.yaml
kind: module
name: scan-base
steps:
  - name: common-step
    type: bash
    command: "common-tool"

# fast.yaml
kind: module
name: scan-fast
extends: scan-base
override:
  params:
    threads:
      default: 100

# aggressive-fast.yaml
kind: module
name: scan-aggressive-fast
extends: scan-fast
override:
  params:
    rate_limit:
      default: 1000
```

### 继承规则

1. **类型必须匹配** - 子工作流和父工作流必须具有相同的 `kind`（module/flow）
2. **循环检测** - 检测并拒绝循环继承链
3. **名称唯一性** - 子工作流的 `name` 覆盖父工作流的名称
4. **文件路径** - 保留子工作流的 `FilePath`（用于错误报告）

## 最佳实践

1. **每个模块一个任务** - 保持模块聚焦
2. **使用 Flow 构建管线** - 通过依赖关系编排
3. **描述性名称** - 使用 `subdomain-enum` 而非 `step1`
4. **文档化参数** - 添加描述
5. **使用标签** - 启用过滤
6. **使用继承** - 创建基工作流和专用变体
7. **优先使用 merge 模式** - 在子工作流中实现细粒度步骤控制

## 下一步

* [步骤类型](step-types) - 所有步骤类型
* [Flows](flows) - 模块编排
* [变量](variables) - 参数和导出
* [控制流](control-flow) - 条件和路由
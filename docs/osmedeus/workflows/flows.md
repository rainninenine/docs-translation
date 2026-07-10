> ## Documentation Index
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# Flow Workflows

> 带依赖关系的多模块编排

Flow 编排多个模块，支持依赖关系和条件执行。

## Basic Flow

```yaml theme={null}
kind: flow
name: basic-recon
description: 基础侦察工作流

params:
  - name: target
    required: true

modules:
  - name: subdomain-enum
    path: modules/subdomain-enum.yaml

  - name: http-probe
    path: modules/http-probe.yaml
    depends_on:
      - subdomain-enum

  - name: screenshot
    path: modules/screenshot.yaml
    depends_on:
      - http-probe
```

## Module Reference Fields

```yaml theme={null}
modules:
  - name: module-name           # 必需：引用名称
    path: path/to/module.yaml   # 必需：模块文件路径
    depends_on:                 # 可选：依赖列表
      - other-module
    condition: 'expression'     # 可选：跳过条件
    params:                     # 可选：参数覆盖
      key: value
```

### name

Flow 内该模块引用的唯一标识符。

```yaml theme={null}
- name: subdomain-enum      # 用于 depends_on 引用
```

### path

模块 YAML 文件的路径（相对于工作流文件夹）。

```yaml theme={null}
- name: nuclei
  path: modules/nuclei-scan.yaml
```

### depends\_on

必须在此模块运行前完成的模块名称列表。

```yaml theme={null}
- name: vuln-scan
  path: modules/vuln.yaml
  depends_on:
    - subdomain-enum
    - http-probe        # 两者必须先完成
```

### condition

在模块执行前评估的 JavaScript 表达式。若为 false，则跳过该模块。

```yaml theme={null}
- name: screenshot
  path: modules/screenshot.yaml
  depends_on: [http-probe]
  condition: 'fileLength("{{Output}}/live-hosts.txt") > 0'
```

常见条件：

```yaml theme={null}
# 文件有内容
condition: 'fileLength("{{Output}}/data.txt") > 0'

# 文件存在
condition: 'fileExists("{{Output}}/targets.txt")'

# 参数检查
condition: '{{enable_vuln_scan}} == "true"'
```

### params

覆盖模块参数。

```yaml theme={null}
- name: nuclei-fast
  path: modules/nuclei-scan.yaml
  params:
    threads: "100"
    severity: "critical,high"
```

## Inline Modules

模块可以直接定义步骤，而无需引用外部 YAML 文件。这对于不需要单独文件的简单、自包含任务非常有用。

```yaml theme={null}
kind: flow
name: quick-recon

modules:
  - name: quick-check
    description: 内联健康检查
    steps:
      - name: ping
        type: bash
        command: ping -c 1 {{Target}}

      - name: http-check
        type: bash
        command: curl -s -o /dev/null -w '%{http_code}' https://{{Target}}
        exports:
          status_code: "{{stdout}}"

  - name: full-scan
    path: modules/full-scan.yaml
    depends_on: [quick-check]
    condition: '{{status_code}} == "200"'
```

### Inline Module Fields

| 字段             | 描述                                       |
| ---------------- | ------------------------------------------ |
| `name`           | 必需。模块引用名称                         |
| `steps`          | 内联步骤（使该模块成为内联模块）           |
| `description`    | 内联模块的描述                             |
| `runner`         | 内联模块的 Runner 类型（`host`, `docker`, `ssh`） |
| `runner_config`  | 内联模块的 Runner 配置                     |

当定义了 `steps` 时，该模块被视为内联模块——无需 `path`。内联模块支持与外部模块文件相同的步骤类型和特性。

### Inline Module with Runner

```yaml theme={null}
modules:
  - name: docker-scan
    description: 在 Docker 中运行 nuclei
    runner: docker
    runner_config:
      image: projectdiscovery/nuclei:latest
      volumes:
        - "{{Output}}:/output"
    steps:
      - name: nuclei
        type: bash
        command: nuclei -u {{Target}} -o /output/nuclei.txt
```

## Dependency Graph

Flow 创建一个有向无环图（DAG）：

```yaml theme={null}
modules:
  - name: A
    path: modules/a.yaml

  - name: B
    path: modules/b.yaml
    depends_on: [A]

  - name: C
    path: modules/c.yaml
    depends_on: [A]

  - name: D
    path: modules/d.yaml
    depends_on: [B, C]
```

执行顺序：

```
    A           # 步骤 1：A 运行
   / \
  B   C         # 步骤 2：B 和 C 并行运行
   \ /
    D           # 步骤 3：B 和 C 完成后 D 运行
```

## Complex Flow Example

```yaml theme={null}
kind: flow
name: full-assessment
description: 完整安全评估

params:
  - name: target
    required: true
  - name: enable_active
    default: "false"
    description: 启用主动扫描

modules:
  # 被动侦察
  - name: subdomain-enum
    path: modules/subdomain-enum.yaml

  - name: dns-enum
    path: modules/dns-enum.yaml

  # 两者可并行运行，均无依赖
  # 执行：subdomain-enum 和 dns-enum 同时运行

  - name: http-probe
    path: modules/http-probe.yaml
    depends_on:
      - subdomain-enum
      - dns-enum
    # 等待两者完成

  - name: screenshot
    path: modules/screenshot.yaml
    depends_on: [http-probe]
    condition: 'fileLength("{{Output}}/live.txt") > 10'

  - name: content-discovery
    path: modules/content-discovery.yaml
    depends_on: [http-probe]

  # 主动扫描（条件性）
  - name: port-scan
    path: modules/port-scan.yaml
    depends_on: [subdomain-enum]
    condition: '{{enable_active}} == "true"'

  - name: vuln-scan
    path: modules/vuln-scan.yaml
    depends_on:
      - http-probe
      - port-scan
    condition: '{{enable_active}} == "true" && fileExists("{{Output}}/live.txt")'

  # 报告生成
  - name: generate-report
    path: modules/report.yaml
    depends_on:
      - screenshot
      - content-discovery
      - vuln-scan
```

## Running Flows

```bash theme={null}
# 基本执行
osmedeus run -f full-assessment -t example.com

# 带参数
osmedeus run -f full-assessment -t example.com -p 'enable_active=true'

# 排除特定模块
osmedeus run -f full-assessment -t example.com -x vuln-scan -x port-scan

# 预演（预览）
osmedeus run -f full-assessment -t example.com --dry-run
```

## Module Exclusion

使用精确匹配（`-x`）或子串匹配（`-X`）跳过特定模块：

```bash theme={null}
# 精确匹配 - 按名称排除特定模块
osmedeus run -f full-assessment -t target -x screenshot -x port-scan

# 模糊匹配 - 排除名称包含子串的模块
osmedeus run -f full-assessment -t target -X nmap -X nuclei
```

| 标志                             | 描述                                         |
| -------------------------------- | -------------------------------------------- |
| `-x, --exclude <module>`        | 按精确名称排除模块（可重复）                 |
| `-X, --fuzzy-exclude <substr>`  | 排除名称包含子串的模块（可重复）             |

被排除的模块视为已完成（依赖关系得到满足）。

## Flow-Level vs Module-Level Params

```yaml theme={null}
# Flow 定义
kind: flow
name: my-flow
params:
  - name: target
    required: true
  - name: threads
    default: "50"           # Flow 级别默认值

modules:
  - name: scan
    path: modules/scan.yaml
    params:
      threads: "{{threads}}"   # 使用 Flow 参数
      wordlist: "/custom.txt"  # 模块特定覆盖
```

解析顺序：

1. 模块引用 `params`（最高优先级）
2. Flow 级别 `params`
3. 模块自身的默认参数（最低优先级）

## Error Handling

默认情况下，如果某个模块失败：

* 依赖该模块的模块将被跳过
* 其他独立分支继续执行

```yaml theme={null}
modules:
  - name: A
    path: modules/a.yaml

  - name: B
    path: modules/b.yaml
    depends_on: [A]
    # 如果 A 失败，B 被跳过

  - name: C
    path: modules/c.yaml
    # C 无论 A 或 B 如何都运行
```

## Best Practices

1. **分组相关模块**
   ```yaml theme={null}
   # 被动侦察组
   - name: subdomain-enum
   - name: dns-enum

   # 主动扫描组
   - name: port-scan
   - name: vuln-scan
   ```

2. **对可选模块使用条件**
   ```yaml theme={null}
   - name: active-scan
     condition: '{{enable_active}} == "true"'
   ```

3. **在处理前检查文件是否存在**
   ```yaml theme={null}
   - name: process-results
     condition: 'fileLength("{{Output}}/data.txt") > 0'
   ```

4. **参数化模块行为**
   ```yaml theme={null}
   - name: nuclei
     params:
       threads: "{{threads}}"
       severity: "{{severity}}"
   ```

5. **添加最终报告模块**
   ```yaml theme={null}
   - name: report
     depends_on: [all, other, modules]
   ```

## Next Steps

* [Variables](variables) - 参数传播
* [Control Flow](control-flow) - 条件详解
* [Step Types](step-types) - 模块步骤类型
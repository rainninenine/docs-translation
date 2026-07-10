> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 变量与导出（Variables and Exports）

> 数据流与参数管理

使用参数和导出管理步骤间的数据流。

## 参数（Parameters）

### 定义参数

```yaml theme={null}
params:
  - name: target
    required: true
    description: 目标域名

  - name: threads
    default: "10"
    description: 线程数

  - name: wordlist
    default: "{{Data}}/wordlists/common.txt"
```

### 使用参数

```yaml theme={null}
steps:
  - name: scan
    type: bash
    command: subfinder -d {{target}} -t {{threads}} -w {{wordlist}}
```

### 传递参数

```bash theme={null}
# 单个参数
osmedeus run -m scan -t example.com -p 'threads=20'

# 多个参数
osmedeus run -m scan -t example.com -p 'threads=20' -p 'wordlist=/path/list.txt'

# 从文件读取
osmedeus run -m scan -t example.com -P params.yaml
```

## 导出（Exports）

导出将值从一个步骤传递到下一个步骤。

### 基本导出

```yaml theme={null}
steps:
  - name: count-lines
    type: bash
    command: wc -l {{Output}}/hosts.txt | cut -d' ' -f1
    exports:
      host_count: "{{stdout}}"

  - name: log-count
    type: function
    function: log_info("Found {{host_count}} hosts")
```

### 导出源

| 源（Source）             | 描述（Description）       |
| ------------------------ | ------------------------- |
| `{{stdout}}`             | 命令标准输出              |
| `{{stderr}}`             | 命令标准错误              |
| `{{exit_code}}`          | 命令退出码                |
| `{{http_status_code}}`   | HTTP 响应状态码           |
| `{{http_response_body}}` | HTTP 响应体               |
| `{{llm_response}}`       | LLM 聊天响应              |

### HTTP 导出

```yaml theme={null}
- name: api-call
  type: http
  url: "https://api.example.com/data"
  method: GET
  exports:
    response_data: "{{http_response_body}}"
    status: "{{http_status_code}}"

- name: process
  type: function
  function: log_info("Status: {{status}}, Data: {{response_data}}")
```

### Agent 导出

```yaml theme={null}
- name: analyze
  type: agent
  query: "Analyze {{Target}} for vulnerabilities"
  max_iterations: 10
  agent_tools:
    - preset: bash
    - preset: read_file
  exports:
    findings: "{{agent_content}}"
    token_usage: "{{agent_total_tokens}}"
```

| 源（Source）                        | 描述（Description）                                      |
| ----------------------------------- | -------------------------------------------------------- |
| `{{agent_content}}`                 | 最终 Agent 响应文本                                      |
| `{{agent_history}}`                 | 完整对话历史（JSON）                                     |
| `{{agent_iterations}}`              | 使用的迭代次数                                           |
| `{{agent_total_tokens}}`            | 消耗的总 Token 数                                        |
| `{{agent_prompt_tokens}}`           | 消耗的提示 Token 数                                      |
| `{{agent_completion_tokens}}`       | 消耗的补全 Token 数                                      |
| `{{agent_tool_results}}`            | 所有工具调用结果（JSON）                                 |
| `{{agent_plan}}`                    | 规划阶段输出（如果设置了 `plan_prompt`）                 |
| `{{agent_goal_results}}`            | 每个目标的结果（用于多目标 `queries`）                   |

### 函数导出

```yaml theme={null}
- name: read-file
  type: function
  function: readFile("{{Output}}/data.txt")
  exports:
    file_content: "{{result}}"

- name: use-content
  type: bash
  command: echo "Content: {{file_content}}"
```

## 变量作用域（Variable Scope）

### 步骤级作用域

导出对所有后续步骤可用：

```yaml theme={null}
steps:
  - name: step1
    type: bash
    command: echo "value1"
    exports:
      var1: "{{stdout}}"

  - name: step2
    type: bash
    command: echo "value2"
    exports:
      var2: "{{stdout}}"

  - name: step3
    type: bash
    command: echo "{{var1}} and {{var2}}"  # 两者均可访问
```

### Foreach 变量作用域

循环变量使用 `[[]]` 语法，仅在循环内部可用：

```yaml theme={null}
- name: process-hosts
  type: foreach
  input: "{{Output}}/hosts.txt"
  variable: host
  step:
    name: scan
    type: bash
    command: nmap [[host]] -o {{Output}}/nmap-[[host]].txt
    # [[host]] = 当前迭代值
    # {{Output}} = 常规模板变量
```

## 解析顺序（Resolution Order）

变量按以下顺序解析：

1. 来自先前步骤的**导出（Exports）**
2. 来自用户输入的**参数（Parameters）**
3. **内置变量（Built-in variables）**（Target、Output 等）
4. **环境变量（Environment variables）**

## 内置变量（Built-in Variables）

Osmedeus 提供一组全面的内置变量，在所有工作流中自动可用。这些变量会被 linter 识别，无需定义。

### 路径变量

| 变量（Variable）            | 描述（Description）                    |
| --------------------------- | -------------------------------------- |
| `{{BaseFolder}}`            | Osmedeus 安装目录                      |
| `{{Binaries}}`              | 工具二进制文件路径                     |
| `{{Data}}`                  | 数据文件路径                           |
| `{{ExternalData}}`          | 外部数据文件路径                       |
| `{{ExternalConfigs}}`       | 外部配置文件路径                       |
| `{{ExternalAgentConfigs}}`  | Agent 配置文件路径                     |
| `{{ExternalAgents}}`        | Agent 脚本路径                         |
| `{{ExternalScripts}}`       | 外部脚本路径                           |
| `{{Workflows}}`             | 工作流目录路径                         |
| `{{MarkdownTemplates}}`     | Markdown 模板路径                      |
| `{{ExternalMarkdowns}}`     | 外部 Markdown 文件路径                 |
| `{{SnapshotsFolder}}`       | 快照存储路径                           |
| `{{Workspaces}}`            | 项目空间目录路径                       |

### 目标变量

| 变量（Variable）    | 描述（Description）                           |
| ------------------- | --------------------------------------------- |
| `{{Target}}`        | 当前扫描目标                                  |
| `{{target}}`        | 当前扫描目标（小写别名）                      |
| `{{TargetFile}}`    | 目标文件路径（用于多目标运行）                |
| `{{TargetSpace}}`   | 清理后的目标（文件系统安全）                  |

### 输出变量

| 变量（Variable）  | 描述（Description）                    |
| ----------------- | -------------------------------------- |
| `{{Output}}`      | 项目空间输出目录                       |
| `{{output}}`      | 项目空间输出目录（小写别名）           |
| `{{Workspace}}`   | 项目空间目录                           |
| `{{workspace}}`   | 项目空间目录（小写别名）               |

### 线程变量

| 变量（Variable）    | 描述（Description）          |
| ------------------- | ---------------------------- |
| `{{threads}}`       | 线程数（基于策略）           |
| `{{Threads}}`       | 线程数（大写别名）           |
| `{{baseThreads}}`   | 基础线程数                   |

### 元数据变量

| 变量（Variable）     | 描述（Description）                           |
| -------------------- | --------------------------------------------- |
| `{{Version}}`        | Osmedeus 版本                                 |
| `{{TaskDate}}`       | 任务日期                                      |
| `{{TaskID}}`         | 唯一任务标识符                                |
| `{{TimeStamp}}`      | Unix 时间戳                                   |
| `{{CurrentTime}}`    | 当前时间                                      |
| `{{Today}}`          | 当前日期（YYYY-MM-DD）                        |
| `{{RandomString}}`   | 随机 6 字符字符串                             |
| `{{ModuleName}}`     | 当前模块名称                                  |
| `{{FlowName}}`       | 父 Flow 名称（直接运行模块时为空）            |
| `{{RunUUID}}`        | 唯一运行标识符（UUID）                        |
| `{{DBRunID}}`        | 数据库运行 ID                                 |

### 状态文件变量

| 变量（Variable）                | 描述（Description）          |
| ------------------------------- | ---------------------------- |
| `{{StateExecutionLog}}`         | 执行日志路径                 |
| `{{StateConsoleLog}}`           | 控制台日志路径               |
| `{{StateCompletedFile}}`        | 完成标记文件路径             |
| `{{StateFile}}`                 | 状态文件路径                 |
| `{{StateWorkflowFile}}`         | 工作流状态文件路径           |
| `{{StateWorkflowFolder}}`       | 工作流状态文件夹路径         |

### 启发式变量

这些变量由自动目标分析填充：

| 变量（Variable）              | 描述（Description）                     |
| ----------------------------- | --------------------------------------- |
| `{{TargetType}}`              | 检测到的目标类型                        |
| `{{TargetRootDomain}}`        | 从目标提取的根域名                      |
| `{{TargetTLD}}`               | 顶级域名                                |
| `{{TargetSLD}}`               | 二级域名                                |
| `{{Org}}`                     | 组织（如果检测到）                      |
| `{{TargetBaseURL}}`           | 目标的基础 URL                          |
| `{{TargetRootURL}}`           | 目标的根 URL                            |
| `{{TargetHostname}}`          | 目标 URL 中的主机名                     |
| `{{TargetHost}}`              | 目标中的主机                            |
| `{{TargetPort}}`              | 目标 URL 中的端口                       |
| `{{TargetPath}}`              | 目标 URL 中的路径                       |
| `{{TargetFileExt}}`           | 目标 URL 中的文件扩展名                 |
| `{{TargetScheme}}`            | URL 协议（http/https）                  |
| `{{TargetIsWildcard}}`        | 目标是否为通配符                        |
| `{{TargetResolvedIP}}`        | 解析的 IP 地址                          |
| `{{TargetStatusCode}}`        | 目标的 HTTP 状态码                      |
| `{{TargetContentLength}}`     | 目标响应的内容长度                      |
| `{{HeuristicsCheck}}`         | 启发式分析结果                          |

### 平台变量

自动检测的环境信息：

| 变量（Variable）                  | 描述（Description）                                    |
| --------------------------------- | ------------------------------------------------------ |
| `{{PlatformOS}}`                  | 操作系统（`linux`、`darwin`、`windows`）               |
| `{{PlatformArch}}`                | CPU 架构（`amd64`、`arm64`）                           |
| `{{PlatformInDocker}}`            | 如果在 Docker 容器内运行则为 `"true"`                  |
| `{{PlatformInKubernetes}}`        | 如果在 Kubernetes Pod 内运行则为 `"true"`              |
| `{{PlatformCloudProvider}}`       | 云提供商名称（`aws`、`gcp`、`azure`、`local`）        |

```yaml theme={null}
steps:
  - name: platform-check
    type: bash
    pre_condition: '{{PlatformOS}} == "linux"'
    command: linux-specific-tool {{Target}}
```

### 事件变量

当工作流由事件触发时可用：

| 变量（Variable）       | 描述（Description）                      |
| ---------------------- | ---------------------------------------- |
| `{{EventEnvelope}}`    | 完整事件信封（JSON）                     |
| `{{EventTopic}}`       | 事件主题（例如 `assets.new`）            |
| `{{EventSource}}`      | 事件来源                                 |
| `{{EventDataType}}`    | 事件数据类型                             |
| `{{EventTimestamp}}`   | 事件时间戳                               |

### 分块变量

用于并行处理大型输入：

| 变量（Variable）    | 描述（Description）           |
| ------------------- | ----------------------------- |
| `{{ChunkIndex}}`    | 当前分块索引                  |
| `{{ChunkSize}}`     | 每个分块的大小                |
| `{{TotalChunks}}`   | 总分块数                      |
| `{{ChunkStart}}`    | 当前分块的起始偏移量          |
| `{{ChunkEnd}}`      | 当前分块的结束偏移量          |

```yaml theme={null}
params:
  - name: threads
    default: "10"

steps:
  - name: first
    type: bash
    command: echo "20"
    exports:
      threads: "{{stdout}}"   # 导出名为 'threads'

  - name: second
    type: bash
    command: run -t {{threads}}  # 使用导出值 (20)，而非参数值 (10)
```

## 嵌套变量（Nested Variables）

变量可以包含其他变量：

```yaml theme={null}
params:
  - name: scan_type
    default: "basic"
  - name: output_path
    default: "{{Output}}/{{scan_type}}"

steps:
  - name: scan
    type: bash
    command: scan -o {{output_path}}/results.txt
    # 解析为：{{Output}}/basic/results.txt
```

## 生成器函数（Generator Functions）

使用 `generator` 字段为参数生成动态值：

```yaml theme={null}
params:
  - name: scan_id
    generator: "uuid()"

  - name: timestamp
    generator: "currentTimestamp()"

  - name: user
    generator: "getEnvVar('USER', 'unknown')"

  - name: run_date
    generator: "currentDate()"
```

`generator` 字段在工作流加载时求值，以生成参数值。

### 可用生成器

| 生成器（Generator）                  | 描述（Description）                              | 示例（Example）                  |
| ------------------------------------ | ------------------------------------------------ | -------------------------------- |
| `uuid()`                             | UUID v4                                          | `a1b2c3d4-...`                   |
| `currentDate(format?)`               | 当前日期（默认：YYYY-MM-DD）                     | `2026-02-17`                     |
| `currentTimestamp()`                 | Unix 时间戳                                      | `1739808000`                     |
| `getEnvVar(key, default?)`           | 环境变量                                         | `getEnvVar('USER', 'unknown')`   |
| `concat(str1, str2, ...)`            | 拼接字符串                                       | `concat('scan-', 'target')`      |
| `randomInt(min?, max?)`              | 随机整数（默认：0-100）                          | `randomInt(1, 1000)`             |
| `randomString(length?)`              | 随机字母数字字符串（默认：16）                   | `randomString(8)`                |
| `execCmd(command)`                   | 执行 shell 命令                                  | `execCmd('whoami')`              |
| `toLower(str)`                       | 转换为小写                                       | `toLower('ABC')`                 |
| `toUpper(str)`                       | 转换为大写                                       | `toUpper('abc')`                 |
| `trim(str)`                          | 去除首尾空白                                     | `trim(' hello ')`                |
| `replace(str, old, new)`             | 替换子串                                         | `replace('a-b', '-', '_')`       |
| `split(str, delim, index?)`          | 分割并获取元素                                   | `split('a,b,c', ',', 1)`         |
| `join(delim, str1, str2, ...)`       | 用分隔符拼接字符串                               | `join('-', 'a', 'b')`            |

## Flow 变量传播（Flow Variable Propagation）

### Flow 到模块

```yaml theme={null}
# Flow
kind: flow
params:
  - name: target
  - name: threads
    default: "50"

modules:
  - name: scan
    path: modules/scan.yaml
    params:
      threads: "{{threads}}"   # 将 Flow 参数传递给模块
```

### Flow 中的模块导出

模块导出不会自动对其他模块可用。请使用共享输出文件：

```yaml theme={null}
# 模块 A 写入
command: subfinder -d {{target}} -o {{Output}}/subs.txt

# 模块 B 读取（depends_on: [A]）
command: httpx -l {{Output}}/subs.txt
```

## 常见模式（Common Patterns）

### 链式处理

```yaml theme={null}
steps:
  - name: enumerate
    type: bash
    command: subfinder -d {{target}} -silent
    exports:
      raw_subs: "{{stdout}}"

  - name: filter
    type: function
    function: |
      split("{{raw_subs}}", "\n")
        .filter(s => s.endsWith(".{{target}}"))
        .join("\n")
    exports:
      filtered_subs: "{{result}}"

  - name: save
    type: bash
    command: echo "{{filtered_subs}}" > {{Output}}/subs.txt
```

### 基于导出的条件判断

```yaml theme={null}
steps:
  - name: count
    type: bash
    command: wc -l < {{Output}}/hosts.txt
    exports:
      count: "{{stdout}}"

  - name: scan-if-hosts
    type: bash
    pre_condition: '{{count}} > 0'
    command: nuclei -l {{Output}}/hosts.txt
```

### 环境变量

```yaml theme={null}
params:
  - name: api_key
    default: "{{getEnvVar('API_KEY', '')}}"

steps:
  - name: api-call
    type: http
    url: "https://api.example.com/scan"
    headers:
      Authorization: "Bearer {{api_key}}"
```

## 最佳实践（Best Practices）

1. **使用描述性导出名称**
   ```yaml theme={null}
   exports:
     subdomain_count: "{{stdout}}"  # 好
     x: "{{stdout}}"                # 差
   ```

2. **记录参数含义**
   ```yaml theme={null}
   params:
     - name: severity
       default: "high,critical"
       description: Nuclei 严重性过滤器
   ```

3. **提供合理的默认值**
   ```yaml theme={null}
   params:
     - name: threads
       default: "10"   # 无需显式参数即可工作
   ```

4. **对大数据使用文件**
   ```yaml theme={null}
   # 好：写入文件
   command: subfinder -d {{target}} -o {{Output}}/subs.txt

   # 避免：大型 stdout 导出
   exports:
     all_subs: "{{stdout}}"  # 可能非常大
   ```

## 下一步

* [控制流（Control Flow）](control-flow) - 在条件中使用导出
* [模板（Templates）](../concepts/templates) - 模板语法
* [函数参考（Functions Reference）](../functions/reference) - 可用函数
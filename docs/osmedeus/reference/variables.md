> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 默认变量参考

> Osmedeus 中模板变量和工具函数的完整参考

# 默认变量参考

Osmedeus 工作流使用模板变量进行动态内容替换。变量用双花括号括起来：`{{variable}}`。

## 模板语法

### 标准变量

标准变量使用 `{{variable}}` 语法，并在步骤执行时求值。

```yaml
command: "nuclei -l {{Output}}/urls.txt -t {{Data}}/templates/"
```

### 次要变量（Foreach）

Foreach 循环使用 `[[variable]]` 语法以避免与标准模板冲突：

```yaml
- name: scan-each
  type: foreach
  input: "{{Output}}/subdomains.txt"
  variable: sub
  step:
    type: bash
    command: "httpx -u [[sub]] >> {{Output}}/httpx.txt"
```

## 内置变量

### 路径变量

| 变量                         | 描述                         | 默认值                                      |
| ---------------------------- | ---------------------------- | ------------------------------------------- |
| `{{BaseFolder}}`             | 基础安装文件夹               | `~/osmedeus-base`                           |
| `{{base_folder}}`            | BaseFolder 的别名            | `~/osmedeus-base`                           |
| `{{Binaries}}`               | 二进制文件路径               | `{{BaseFolder}}/external-binaries`          |
| `{{Data}}`                   | 数据文件路径                 | `{{BaseFolder}}/data`                       |
| `{{ExternalData}}`           | Data 的别名                  | `{{BaseFolder}}/data`                       |
| `{{ExternalConfigs}}`        | 外部配置文件路径             | `{{BaseFolder}}/configs`                    |
| `{{ExternalScripts}}`        | 外部脚本路径                 | `{{BaseFolder}}/scripts`                    |
| `{{ExternalAgentConfigs}}`   | Agent 配置文件路径           | `{{BaseFolder}}/external-agent-configs`     |
| `{{ExternalAgents}}`         | ExternalAgentConfigs 的别名  | `{{BaseFolder}}/external-agent-configs`     |
| `{{Workflows}}`              | 工作流路径                   | `{{BaseFolder}}/workflows`                  |
| `{{MarkdownTemplates}}`      | Markdown 报告模板路径        | `{{BaseFolder}}/markdown-report-templates`  |
| `{{ExternalMarkdowns}}`      | MarkdownTemplates 的别名     | `{{BaseFolder}}/markdown-report-templates`  |
| `{{Workspaces}}`             | 项目空间路径（可通过 -W 标志覆盖） | `~/workspaces-osmedeus`                     |
| `{{SnapshotsFolder}}`        | 快照路径                     | `{{BaseFolder}}/snapshots`                  |

### 目标变量

| 变量            | 描述                         | 示例                          |
| --------------- | ---------------------------- | ----------------------------- |
| `{{Target}}`    | 当前扫描目标                 | `example.com`                 |
| `{{target}}`    | Target 的别名                | `example.com`                 |
| `{{TargetFile}}`| 包含目标的文件（-T 标志）    | `/path/to/targets.txt`        |
| `{{TargetSpace}}`| 清理后的目标（项目空间名称） | `example_com`                 |
| `{{Output}}`    | 目标的输出目录               | `{{Workspaces}}/example_com`  |
| `{{Workspace}}` | TargetSpace 的别名           | `example_com`                 |
| `{{workspace}}` | Workspace 的别名             | `example_com`                 |

### 目标启发式变量（自动检测）

这些变量根据目标类型检测自动填充。设置 `--heuristics none` 可禁用。

| 变量                | 描述         | 适用类型                               |
| ------------------- | ------------ | -------------------------------------- |
| `{{TargetType}}`    | 检测到的类型 | `url`, `domain`, `ip`, `file`, `unknown` |
| `{{HeuristicsCheck}}`| 启发式级别   | `basic`（默认）, `deep`, `none`        |

**URL 目标变量**（当 `{{TargetType}}` 为 `url` 时）：

| 变量                    | 描述                     | 示例                        |
| ----------------------- | ------------------------ | --------------------------- |
| `{{TargetBaseURL}}`     | 不带路径的基础 URL       | `https://example.com:8080`  |
| `{{TargetRootURL}}`     | 根 URL（协议+主机）      | `https://example.com`       |
| `{{TargetHostname}}`    | URL 中的主机名           | `example.com`               |
| `{{TargetRootDomain}}`  | 根域名                   | `example.com`               |
| `{{TargetTLD}}`         | 顶级域名                 | `com`                       |
| `{{TargetSLD}}`         | 二级域名                 | `example`                   |
| `{{Org}}`               | TargetSLD 的别名         | `example`                   |
| `{{TargetHost}}`        | 带端口的主机             | `example.com:8080`          |
| `{{TargetPort}}`        | 端口号                   | `8080`                      |
| `{{TargetPath}}`        | URL 路径                 | `/api/v1`                   |
| `{{TargetFileExt}}`     | 文件扩展名               | `html`                      |
| `{{TargetScheme}}`      | URL 协议                 | `https`                     |
| `{{TargetStatusCode}}`  | HTTP 状态码（如检测到）  | `200`                       |
| `{{TargetContentLength}}`| 内容长度（如检测到）    | `1234`                      |

**域名目标变量**（当 `{{TargetType}}` 为 `domain` 时）：

| 变量                 | 描述                         | 示例           |
| -------------------- | ---------------------------- | -------------- |
| `{{TargetRootDomain}}`| 根域名                       | `example.com`  |
| `{{TargetTLD}}`      | 顶级域名                     | `com`          |
| `{{TargetSLD}}`      | 二级域名                     | `example`      |
| `{{Org}}`            | TargetSLD 的别名             | `example`      |
| `{{TargetIsWildcard}}`| 是否为通配符子域名           | `true`, `false`|
| `{{TargetResolvedIP}}`| 解析的 IP 地址（如可用）     | `93.184.216.34`|

**IP 目标变量**（当 `{{TargetType}}` 为 `ip` 时）：

| 变量                 | 描述           | 示例           |
| -------------------- | -------------- | -------------- |
| `{{TargetRootDomain}}`| 原始 IP 值     | `192.168.1.1`  |

### 平台检测变量

| 变量                        | 描述                 | 示例值                           |
| --------------------------- | -------------------- | -------------------------------- |
| `{{PlatformOS}}`            | 操作系统             | `linux`, `darwin`, `windows`     |
| `{{PlatformArch}}`          | CPU 架构             | `amd64`, `arm64`                 |
| `{{PlatformInDocker}}`      | 是否在 Docker 容器中 | `true`, `false`                  |
| `{{PlatformInKubernetes}}`  | 是否在 Kubernetes Pod 中 | `true`, `false`              |
| `{{PlatformCloudProvider}}` | 检测到的云供应商     | `aws`, `gcp`, `azure`, `local`   |

### 线程变量

| 变量            | 描述                     | 默认值 |
| --------------- | ------------------------ | ------ |
| `{{threads}}`   | 线程数（基于策略）       | `10`   |
| `{{baseThreads}}`| 基础线程数（threads / 2）| `5`    |

### 元数据变量

| 变量             | 描述                                         | 示例                                  |
| ---------------- | -------------------------------------------- | ------------------------------------- |
| `{{Version}}`    | Osmedeus 版本                                | `5.0.0`                               |
| `{{ModuleName}}` | 当前工作流/模块名称                          | `subdomain-enum`                      |
| `{{workflow}}`   | ModuleName 的别名                            | `subdomain-enum`                      |
| `{{FlowName}}`   | 父 Flow 名称（直接运行模块时为空）           | `recon-flow`                          |
| `{{RunUUID}}`    | 当前运行执行的 UUID                          | `550e8400-e29b-41d4-a716-446655440000`|
| `{{run_uuid}}`   | RunUUID 的别名                               | `550e8400-e29b-41d4-a716-446655440000`|
| `{{DBRunID}}`    | 用于数据库外键的整数 Run.ID                  | `42`                                  |
| `{{TaskDate}}`   | 任务日期                                     | `2025-01-15`                          |
| `{{Today}}`      | 当前日期                                     | `2025-01-15`                          |
| `{{TimeStamp}}`  | Unix 时间戳                                  | `1705312800`                          |
| `{{CurrentTime}}`| 当前时间（ISO 8601）                         | `2025-01-15T10:00:00`                 |
| `{{RandomString}}`| 随机 6 字符小写字符串                        | `xkmprq`                              |

### 常量

| 变量          | 描述                 | 示例             |
| ------------- | -------------------- | ---------------- |
| `{{DefaultUA}}`| 默认 User-Agent 字符串 | `Mozilla/5.0 ...`|

### 状态文件变量

| 变量                    | 描述                 |
| ----------------------- | -------------------- |
| `{{StateExecutionLog}}` | 执行日志路径         |
| `{{StateConsoleLog}}`   | 控制台日志路径       |
| `{{StateCompletedFile}}`| run-completed.json 路径 |
| `{{StateFile}}`         | run-state.json 路径  |
| `{{StateWorkflowFile}}` | run-workflow.yaml 路径 |
| `{{StateWorkflowFolder}}`| run-modules 文件夹路径 |

### 分块模式变量

用于使用 `--chunk` 标志的分布式扫描：

| 变量            | 描述               |
| --------------- | ------------------ |
| `{{ChunkIndex}}`| 当前分块索引       |
| `{{ChunkSize}}` | 每块项目数         |
| `{{TotalChunks}}`| 总块数             |
| `{{ChunkStart}}`| 起始位置           |
| `{{ChunkEnd}}`  | 结束位置           |

### 事件触发变量

这些变量仅适用于事件触发的工作流：

| 变量               | 描述                         | 示例                          |
| ------------------ | ---------------------------- | ----------------------------- |
| `{{EventEnvelope}}`| 完整事件信封（JSON 字符串）  | `{"topic":"assets.new",...}`  |
| `{{EventTopic}}`   | 事件主题                     | `assets.new`                  |
| `{{EventSource}}`  | 事件来源                     | `nuclei`                      |
| `{{EventDataType}}`| 事件数据类型                 | `asset`                       |
| `{{EventTimestamp}}`| 事件时间戳                   | `2025-01-15T10:00:00Z`        |
| `{{EventData}}`    | 事件数据负载（JSON 字符串）  | `{"url":"https://...",...}`   |

## 工具函数

工具函数通过 Goja JavaScript 运行时执行，可在函数步骤或模板表达式中使用。

### 文件操作

| 函数                                      | 返回类型   | 描述                         |
| ----------------------------------------- | ---------- | ---------------------------- |
| `fileExists(path)`                        | bool       | 检查文件是否存在             |
| `fileLength(path)`                        | int        | 统计文件中非空行数           |
| `dirLength(path)`                         | int        | 统计目录中条目数             |
| `fileContains(path, pattern)`             | bool       | 检查文件是否包含模式         |
| `regexExtract(path, pattern)`             | \[]string  | 提取文件中匹配模式的行       |
| `readFile(path)`                          | string     | 读取整个文件内容             |
| `readLines(path)`                         | \[]string  | 将文件读取为行数组           |
| `removeFile(path)`                        | bool       | 删除文件                     |
| `removeFolder(path)`                      | bool       | 递归删除文件夹               |
| `rm_rf(path)`                             | bool       | 递归删除文件或文件夹         |
| `remove_all_except(folder, keep)`         | bool       | 删除除 keep_file 外的所有内容 |
| `createFolder(path)`                      | bool       | 递归创建文件夹               |
| `appendFile(dest, source)`                | bool       | 将源文件追加到目标文件       |
| `moveFile(source, dest)`                  | bool       | 移动/重命名文件              |
| `glob(pattern)`                           | \[]string  | 列出匹配 glob 模式的文件     |
| `grep_string(source, str)`                | string     | 返回包含字符串的行           |
| `grep_regex(source, pattern)`             | string     | 返回匹配正则表达式的行       |
| `grep_string_to_file(dest, source, str)`  | bool       | 将匹配行写入文件             |
| `grep_regex_to_file(dest, source, pattern)`| bool       | 将匹配行写入文件             |
| `remove_blank_lines(path)`                | bool       | 原地移除空行                 |

### 字符串操作

| 函数                                | 返回类型   | 描述                         |
| ----------------------------------- | ---------- | ---------------------------- |
| `trim(str)`                         | string     | 去除首尾空白                 |
| `split(str, delim)`                 | \[]string  | 按分隔符分割                 |
| `join(arr, delim)`                  | string     | 用分隔符连接                 |
| `replace(str, old, new)`            | string     | 替换所有出现                 |
| `contains(str, substr)`             | bool       | 检查是否包含子串             |
| `startsWith(str, prefix)`           | bool       | 检查是否以前缀开头           |
| `endsWith(str, suffix)`             | bool       | 检查是否以后缀结尾           |
| `toLowerCase(str)`                  | string     | 转换为小写                   |
| `toUpperCase(str)`                  | string     | 转换为大写                   |
| `match(str, pattern)`               | bool       | 检查正则匹配                 |
| `regex_match(pattern, str)`         | bool       | 检查正则匹配（模式在前）     |
| `cut_with_delim(input, delim, field)`| string     | 提取字段（从1开始索引）      |
| `normalize_path(input)`             | string     | 将特殊字符替换为 _           |
| `clean_sub(path, target?)`          | bool       | 清理并去重子域名             |

### 类型转换

| 函数            | 返回类型 | 描述             |
| --------------- | -------- | ---------------- |
| `parseInt(str)` | int      | 将字符串解析为整数 |
| `parseFloat(str)`| float    | 将字符串解析为浮点数 |
| `toString(val)` | string   | 转换为字符串     |
| `toBoolean(val)`| bool     | 转换为布尔值     |

### 工具

| 函数              | 返回类型 | 描述                 |
| ----------------- | -------- | -------------------- |
| `len(val)`        | int      | 获取字符串/数组长度  |
| `isEmpty(val)`    | bool     | 检查是否为空         |
| `isNotEmpty(val)` | bool     | 检查是否非空         |
| `printf(message)` | void     | 打印到标准输出       |
| `cat_file(path)`  | void     | 打印文件内容         |
| `exit(code)`      | void     | 以指定代码退出       |
| `exec_cmd(command)`| string   | 执行 bash 命令       |
| `sleep(seconds)`  | void     | 暂停执行             |

### 日志

| 函数               | 返回类型 | 描述                   |
| ------------------ | -------- | ---------------------- |
| `log_debug(message)`| void     | 以 \[DEBUG] 前缀记录日志 |
| `log_info(message)` | void     | 以 \[INFO] 前缀记录日志  |
| `log_warn(message)` | void     | 以 \[WARN] 前缀记录日志  |
| `log_error(message)`| void     | 以 \[ERROR] 前缀记录日志 |

### HTTP

| 函数                                    | 返回类型 | 描述           |
| --------------------------------------- | -------- | -------------- |
| `httpRequest(url, method, headers, body)`| object   | 发起 HTTP 请求 |
| `http_get(url)`                         | object   | HTTP GET 请求  |
| `http_post(url, body)`                  | object   | HTTP POST 请求 |

### 生成

| 函数                 | 返回类型 | 描述               |
| -------------------- | -------- | ------------------ |
| `randomString(length)`| string   | 随机字母数字字符串 |
| `uuid()`             | string   | 生成 UUID v4       |

### 编码

| 函数              | 返回类型 | 描述         |
| ----------------- | -------- | ------------ |
| `base64Encode(str)`| string   | 编码为 base64 |
| `base64Decode(str)`| string   | 从 base64 解码 |

### 数据查询

| 函数                      | 返回类型 | 描述                     |
| ------------------------- | -------- | ------------------------ |
| `jq(jsonData, query)`     | any      | 使用 jq 语法提取数据     |
| `jq_from_file(path, query)`| any      | 从 JSON 文件使用 jq 查询 |

### 通知

| 函数                              | 返回类型 | 描述             |
| --------------------------------- | -------- | ---------------- |
| `sendTelegram(message)`           | void     | 发送 Telegram 消息 |
| `sendSlack(message)`              | void     | 发送 Slack 消息   |
| `sendDiscord(message)`            | void     | 发送 Discord 消息 |
| `sendWebhook(url, message)`       | void     | 发送 Webhook 消息 |
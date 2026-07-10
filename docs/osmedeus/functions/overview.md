> ## Documentation Index
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 函数概述

Osmedeus 通过 Goja JavaScript 运行时提供 190 多个实用函数。函数可在工作流步骤、条件中使用，也可通过 CLI 或 API 进行评估。

## 用法

### 在步骤中

```yaml theme={null}
- name: log-start
  type: function
  function: log_info("Scanning {{target}}")

- name: check-file
  type: function
  functions:
    - log_info("Checking files")
    - file_exists("{{Output}}/data.txt")
```

### 在条件中

```yaml theme={null}
- name: scan
  type: bash
  pre_condition: 'file_length("{{Output}}/hosts.txt") > 0'
  command: nuclei -l {{Output}}/hosts.txt
```

### 在 Flow 条件中

```yaml theme={null}
modules:
  - name: vuln-scan
    path: modules/vuln.yaml
    condition: 'file_exists("{{Output}}/live.txt")'
```

## 函数步骤类型

### 单个函数

```yaml theme={null}
- name: log
  type: function
  function: log_info("Message")
```

### 多个函数（顺序执行）

```yaml theme={null}
- name: setup
  type: function
  functions:
    - log_info("Step 1")
    - log_info("Step 2")
    - log_info("Step 3")
```

### 并行函数

```yaml theme={null}
- name: parallel-checks
  type: function
  parallel_functions:
    - file_length("{{Output}}/file1.txt")
    - file_length("{{Output}}/file2.txt")
    - file_length("{{Output}}/file3.txt")
```

## 返回值

函数返回的值可用于：

### 在导出中使用

```yaml theme={null}
- name: count-lines
  type: function
  function: file_length("{{Output}}/hosts.txt")
  exports:
    host_count: "{{result}}"
```

### 在条件中使用

```yaml theme={null}
- name: scan
  type: bash
  pre_condition: 'file_length("{{Output}}/hosts.txt") > 0'
  command: scan {{Output}}/hosts.txt
```

### 在决策路由中使用

```yaml theme={null}
- name: check
  type: function
  function: file_exists("{{Output}}/critical.txt")
  exports:
    has_critical: "{{result}}"
  decision:
    switch: "{{has_critical}}"
    cases:
      "true": { goto: handle-critical }
    default: { goto: continue-normal }
```

## CLI 评估

### 基本评估

```bash theme={null}
# 列出所有函数
osmedeus func list

# 评估一个函数
osmedeus func e 'file_exists("/path/to/file")'

# 带目标变量
osmedeus func e 'log_info("Scanning " + target)' -t example.com

# 带自定义参数
osmedeus func e 'log_info(prefix + target)' -t example.com --params 'prefix=test_'
```

### 脚本源优先级

CLI 按以下顺序确定要执行的脚本：

1. `--function-file` - 从文件读取脚本
2. `-f/--function` - 函数名称，剩余参数作为参数
3. 位置参数 - `e` 或 `eval` 后的直接表达式
4. `-e/--eval` - 通过标志传递的脚本
5. `--stdin` - 从标准输入读取

### 批量处理

从文件处理多个目标：

```bash theme={null}
# 从文件处理目标
osmedeus func e 'log_info("Processing: " + target)' -T targets.txt

# 带并发
osmedeus func e 'http_get("https://" + target)' -T targets.txt -c 10

# 使用函数文件
osmedeus func e --function-file check.js -T targets.txt -c 5

# 带参数
osmedeus func e 'log_info(prefix + target)' -T targets.txt --params 'prefix=test_' -c 5
```

### 函数列表选项

```bash theme={null}
# 列出所有函数
osmedeus func list

# 搜索/过滤函数
osmedeus func list -s "event"
osmedeus func list -s "file"

# 显示示例
osmedeus func list --example

# 自定义列宽
osmedeus func list --width 80
```

## API 评估

通过 REST API 评估函数：

```bash theme={null}
curl -X POST http://localhost:8002/osm/api/functions/eval \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"script": "file_length(\"/path/to/file\")"}'
```

列出可用函数：

```bash theme={null}
curl http://localhost:8002/osm/api/functions/list \
  -H "Authorization: Bearer $TOKEN"
```

## 上下文变量

函数可以访问执行上下文：

```javascript theme={null}
// 内置变量可用
log_info("Target: " + target)
log_info("Output: " + "{{Output}}")

// 来自先前步骤的导出
log_info("Previous result: " + "{{previous_export}}")
```

## 错误处理

函数失败不会停止工作流执行，除非配置了错误处理：

```yaml theme={null}
- name: risky-function
  type: function
  function: read_file("/possibly/missing/file.txt")
  on_error:
    - action: log
      message: "Function failed"
    - action: continue
```

## 函数类别

Osmedeus 提供 190 多个函数，分为 34 个类别：

| 类别                  | 描述                                   | 数量 |
| --------------------- | -------------------------------------- | ---- |
| **File**              | 文件/目录操作，grep，glob              | 22   |
| **String**            | 字符串操作，正则匹配                   | 21   |
| **Type Conversion**   | 类型之间的解析/转换                    | 4    |
| **Type Detection**    | 检测输入类型（文件、URL、IP 等）       | 7    |
| **Utility**           | 通用工具（len、exec、sleep）           | 11   |
| **Logging**           | 带级别前缀的日志消息                   | 4    |
| **Color Printing**    | 彩色终端输出                           | 4    |
| **Runtime Variables** | 获取/设置运行时变量                    | 2    |
| **HTTP**              | HTTP 请求和 IP 解析                    | 4    |
| **LLM**               | LLM 交互和对话                         | 3    |
| **Generation**        | 随机字符串和 UUID                      | 2    |
| **Encoding**          | Base64 编码/解码                       | 2    |
| **Data Query**        | JQ 风格的 JSON 查询                    | 2    |
| **Notification**      | Telegram 和 webhook 通知               | 8    |
| **Event Generation**  | 结构化事件生成                         | 2    |
| **CDN/Storage**       | 云存储操作（兼容 S3）                  | 11   |
| **Unix Commands**     | sort、wget、git、tar 等命令的封装      | 11   |
| **Archive (Go)**      | 纯 Go 实现的 zip/unzip                 | 3    |
| **Snapshot**          | 项目空间导出/导入为 ZIP 归档           | 2    |
| **Diff**              | 文件比较和差异提取                     | 1    |
| **Output**            | 保存内容，JSONL/CSV 转换               | 6    |
| **URL Processing**    | URL 去重、过滤和解析                   | 6    |
| **Markdown**          | Markdown 渲染和转换                    | 6    |
| **Database**          | 资产/漏洞导入、查询、统计              | 50   |
| **SARIF**             | SARIF 解析和数据库导入                 | 2    |
| **Nmap/Port**         | 端口扫描和结果导入                     | 3    |
| **Installer**         | 通过 go-getter/Nix 下载包              | 4    |
| **Environment**       | 环境变量操作                           | 2    |
| **Tmux**              | 通过 tmux 的后台进程管理               | 5    |
| **SSH/Sync**          | 远程执行和文件同步                     | 5    |
| **Script Execution**  | Python 和 TypeScript 执行              | 4    |
| **Agent/Distributed** | ACP 代理和分布式执行                   | 3    |
| **Module Control**    | 跳过模块和运行模块/Flow                | 3    |
| **Authentication**    | Sudo 身份验证                          | 1    |

## 最佳实践

1. **在条件中使用函数**
   ```yaml theme={null}
   pre_condition: 'file_exists("{{Output}}/input.txt")'
   ```

2. **记录有意义的消息**
   ```yaml theme={null}
   function: log_info("Found " + host_count + " hosts for {{Target}}")
   ```

3. **导出函数结果**
   ```yaml theme={null}
   exports:
     line_count: "{{result}}"
   ```

4. **优雅处理缺失文件**
   ```yaml theme={null}
   pre_condition: 'file_exists("{{Output}}/data.txt")'
   ```

5. **使用适当的日志级别**
   * `log_debug` 用于详细调试
   * `log_info` 用于信息性消息
   * `log_warn` 用于警告
   * `log_error` 用于错误

6. **利用批量处理进行测试**
   ```bash theme={null}
   # 针对多个目标测试函数
   osmedeus func e 'http_get("https://" + target + "/api/health")' -T domains.txt -c 20
   ```

## 后续步骤

* [函数参考](reference) - 完整函数列表及签名
* [控制流](/workflows/control-flow) - 使用条件和决策
* [变量](/workflows/variables) - 导出和参数
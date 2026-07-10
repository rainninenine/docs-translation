# 文档API参考入门输出与报告复制页面Vigolium 支持多种输出格式，用于扫描结果、内容发现数据和爬取输出。本指南涵盖可用格式、结果结构以及如何查询已存储的漏洞发现。

复制页面

## 输出格式

`--format` 标志控制输出格式。共有五种格式可用，且可组合使用（例如 `--format jsonl,html`）：

### 控制台（默认）

```bash
vigolium scan --target https://example.com
```

人类可读的终端输出，带有颜色编码的严重性级别。漏洞发现会在发现时立即打印，扫描结束时显示汇总表。当未指定 `--format` 标志时，此为默认格式。

严重性颜色：

- **严重**: 红色
- **高**: 橙色/黄色
- **中**: 黄色
- **低**: 蓝色
- **信息**: 灰色

### JSONL

```bash
vigolium scan --target https://example.com --format jsonl
```

每行一个 JSON 对象，机器可读。每行是一个独立的 JSON 文档，表示单个漏洞发现或事件。此格式非常适合通过管道传递给 `jq`、导入 SIEM 或使用自定义脚本处理。

与 `jq` 配合使用的示例：

```bash
vigolium scan --target https://example.com --format jsonl | jq 'select(.severity == "high")'
```

对于 CI/CD 管道，`--ci-output-format` 是一个简写，强制输出干净的 JSONL，不包含横幅或颜色代码，可直接安全地用于构建脚本解析：

```bash
vigolium scan --stateless -t https://example.com --ci-output-format -o findings
```

完整的管道示例请参见 CI/CD 集成。

### HTML

```bash
vigolium scan --target https://example.com --format html -o report.html
```

使用嵌入式 ag-grid 表格的交互式 HTML 报告。该报告是一个独立的 HTML 文件，支持排序、过滤和搜索功能。使用 HTML 格式时必须指定 `-o/--output` 标志。

HTML 格式支持以下内容：

- 扫描结果（漏洞发现）
- 内容发现阶段输出（发现的 URL 和端点）
- 爬取阶段输出（爬取的页面）

### SQLite

```bash
vigolium scan --stateless --target https://example.com --format sqlite -o scan
```

将运行时的独立数据库转储到 `<output>.sqlite`（通过 SQLite VACUUM INTO），生成一个独立的文件，之后可使用 `vigolium finding/traffic` 重新打开（参见读取独立导出）。别名：`sqlite3`、`db`。

需要 `-S/--stateless` 和 `-o/--output`。无状态运行使用每次运行的临时数据库，因此“本次扫描的 SQLite”是明确定义的；持久化运行会写入共享的项目数据库，此时会存在歧义（请使用 `vigolium export`）。

可与其他格式组合：`--format sqlite,html -o scan` 会同时写入 `scan.sqlite` 和 `scan.html`。

在 `--split-by-host` 下，每个主机的文件命名为 `<base>-<host>.sqlite`。

### 文件系统（fs）

```bash
vigolium scan --target https://example.com --format fs -o run
```

写入一个扁平的、可浏览的文件系统树，而不是单个文件——这样您（或编码 Agent）可以使用普通的 `ls`/`grep`/`jq` 来检查扫描结果，无需数据库。两个同级目录会基于 `-o` 基础路径（省略 `-o` 时默认为当前目录下的 `vigolium`）写入，因此 `-o run` 会生成 `run-traffic/` 和 `run-findings/`：

```
run-traffic/
  index.json                 # [{id,host,path,method,url,status,content_type,bytes,finding}, …]
  <host>/0001.req            # "@target https://<host>" 行 + 原始请求（可重放）
  <host>/0001.resp.headers   # 状态行 + 响应头
  <host>/0001.resp.body      # 响应体，gzip 解码以便 grep 干净
run-findings/
  index.json                 # [{id,host,path,severity,confidence,module,title,url,traffic}, …]
  <host>/0001.md             # 漏洞发现，交叉链接到 ../../run-traffic/<host>/0001.req
```

每个主机的 ID 是零填充的，并按 `sent_at` 顺序分配，因此重新导出是可重现的。

每个 `<id>.req` 以 `@target <scheme>://<authority>` 行开头，后跟原始请求——去掉第一行即可直接重放。

`index.json` 是入口点：对其执行一次 `jq` 即可将每个 ID 映射到其 url/status 以及包含字节的文件。在流量行上，`finding` 字段携带与该请求相关的任何漏洞发现的最高严重性；每个漏洞发现的 `.md` 文件直接链接到证明它的 `.req/.resp.*` 文件。

可用于 `vigolium export`、`vigolium db export`（遵循其过滤器）以及 `scan`/`scan-url`/`scan-request`/`run`——无论是否使用 `-S/--stateless`。

遵循 `--omit-response`（删除 `.resp.*` 文件）。`--split-by-host` 在此处无效，因为 fs 已经按主机分割。

数据导入服务器可以使用 `vigolium server --mirror-fs <dir>` 实时生成相同的树——参见运行服务器。

## 严重性等级

漏洞发现使用五个严重性级别进行分类：

| 严重性 | 描述 |
|:-------|:-----|
| 严重 | 可被利用且影响严重的漏洞（例如 RCE、带数据泄露的 SQL 注入） |
| 高 | 可能导致数据泄露或未授权访问的重大漏洞 |
| 中 | 需要特定条件才能利用或影响有限的漏洞 |
| 低 | 安全影响最小的次要问题 |
| 信息 | 信息性发现，例如技术指纹或配置细节 |

## 置信度等级

每个漏洞发现包含一个置信度级别，指示检测的可靠性：

| 置信度 | 描述 |
|:-------|:-----|
| 确定 | 有证据确认。扫描器通过直接证据验证了漏洞（例如，反射的载荷被执行、数据被提取）。 |
| 坚实 | 强有力的证据支持该发现。多个指标确认了问题，但未获得直接利用证明。 |
| 推测 | 基于启发式或模式匹配。该发现可能是误报，应手动验证。 |

## 漏洞发现结构

每个漏洞发现包含以下字段：

| 字段 | 描述 |
|:-----|:-----|
| Module | 产生该漏洞发现的扫描器模块（例如 `xss-reflected`、`sqli-error-based`） |
| Severity | 严重、高、中、低或信息 |
| Confidence | 确定、坚实或推测 |
| URL | 检测到漏洞的目标 URL |
| Parameter | 被测试的特定参数或插入点（如果适用） |
| Evidence | 漏洞证明、响应摘录、载荷或其他确认数据 |
| Description | 漏洞及其潜在影响的人类可读解释 |

## 保存输出

### 使用 `-o/--output` 标志

将输出直接写入文件：

```bash
# 保存 JSONL 输出
vigolium scan --target https://example.com --format jsonl -o results.jsonl

# 保存 HTML 报告
vigolium scan --target https://example.com --format html -o report.html

# 保存控制台输出
vigolium scan --target https://example.com -o results.txt
```

### 管道传递 JSONL

JSONL 输出可以通过管道传递给其他工具进行处理：

```bash
# 过滤高和严重漏洞发现
vigolium scan --target https://example.com --format jsonl | jq 'select(.severity == "high" or .severity == "critical")'

# 按严重性统计漏洞发现数量
vigolium scan --target https://example.com --format jsonl | jq -s 'group_by(.severity) | map({severity: .[0].severity, count: length})'

# 仅提取包含漏洞发现的 URL
vigolium scan --target https://example.com --format jsonl | jq -r '.url'
```

## 内容发现和爬取输出

内容发现和爬取阶段会与扫描漏洞发现一起产生各自的输出。

### 内容发现输出

内容发现输出包括通过基于字典的内容发现、Wayback Machine 数据和 JavaScript 分析找到的 URL 和端点。每个发现的 URL 都会报告其 HTTP 状态码和响应元数据。

```bash
# 仅运行内容发现并保存结果
vigolium scan --target https://example.com --only discovery --format html -o discovery-report.html
```

### 爬取输出

爬取输出包括基于浏览器的爬虫找到的页面，以及爬取过程中发现的表单、链接和动态内容。

```bash
# 仅运行爬取并保存结果
vigolium scan --target https://example.com --only spidering --format html -o spider-report.html
```

两个阶段都支持所有三种输出格式（控制台、JSONL、HTML）。

## OAST 交互

带外应用安全测试（OAST）漏洞发现来自 DNS 和 HTTP 回调交互。当扫描器载荷触发对 OAST 服务器的带外请求时，该交互会与原始测试用例关联。

OAST 漏洞发现在输出中显示：

- 触发带外交互的原始请求
- 交互类型（DNS 查找、HTTP 请求）
- 时间信息（何时收到回调）
- 将交互链接到特定载荷的关联数据

OAST 交互可能在初始扫描阶段完成后才到达，因为某些带外触发器具有延迟执行。Vigolium 会在扫描后等待一段可配置的时间，以收集延迟到达的回调。

如果出站 DNS 或 HTTP 被防火墙阻止，基于 OAST 的检测将无法工作。扫描器仍会通过其他检测方法产生漏洞发现，OAST 只是增加了额外的带外检测层。

## 从数据库查询结果

所有扫描数据都存储在数据库中（默认使用 SQLite）。您可以使用 CLI 命令查询存储的结果，而无需重新运行扫描。

### 列出漏洞发现

```bash
# 列出所有漏洞发现
vigolium finding list

# 列出特定项目的漏洞发现
vigolium finding list --project my-project

# 按最低严重性和置信度过滤（添加彩色 CONFIDENCE 列）
vigolium finding list --min-severity high
vigolium finding list --confidence firm
vigolium finding list --confidence certain,firm
```

`--confidence` 仅保留置信度匹配逗号分隔级别（certain、firm、tentative）之一的漏洞发现，并渲染一个彩色的 CONFIDENCE 列，与 `--min-severity` 互补。

### 列出流量

```bash
# 列出记录的 HTTP 流量
vigolium traffic list

# 列出特定项目的流量
vigolium traffic list --project my-project
```

结果限定在活动项目范围内。使用 `--project-name/--project-uuid` 定位特定项目，或使用 `eval $(vigolium project use <uuid>)` 为您的 shell 设置默认值。有关多租户隔离和项目范围的详细信息，请参见项目与多租户隔离参考。

### 读取独立导出

`finding` 和 `traffic` 可以直接读取文件，而不是项目数据库，这对于检查 `--format jsonl` 导出或来自另一台机器的外部 `.sqlite` 文件非常方便。传递 `-S/--stateless` 以及 `--db <file>`：

```bash
# 使用所有常规过滤器和排序浏览扫描的 JSONL 导出
vigolium finding -S --db ./scan.jsonl --min-severity medium
vigolium traffic -S --db ./scan.jsonl --status 500 -n 20

# 独立的 .sqlite（例如来自 --format sqlite）也可以
vigolium finding -S --db ./scan.sqlite --json --with-records
```

`-S/--stateless` 关闭项目范围，因此文件中的每一行都会显示，无论其携带的 `project_uuid` 是什么。不会向项目数据库写入任何内容（JSONL 源会加载到临时的内存 SQLite 中）。源类型通过扩展名自动检测，如果失败则进行头部嗅探（`.jsonl/.ndjson` 与 SQLite 格式 3 魔术字节 / `.sqlite/.sqlite3/.db`）。

### 将漏洞发现或记录渲染为 Markdown

`--markdown` 将选定的漏洞发现/记录以 Markdown 格式打印到标准输出——证据以及请求/响应（放在围栏 http 块中）。通过管道传递给文件或像 `glow` 这样的查看器，并与 `--id`、模糊搜索词或 `-n 1` 结合使用以聚焦单个项目：

```bash
vigolium finding xss --markdown > findings.md
vigolium finding -S --db ./scan.jsonl --id 42 --markdown
vigolium traffic -S --db ./scan.jsonl search-term -n 1 --markdown
```

在 `-S/--stateless` 下，添加 `--compact` 可将响应窗口限制在漏洞发现匹配（`matched_at` / `extracted_results`）周围——或将记录的主体截断为预览——这样长页面不会淹没控制台。没有 `--compact` 时，主体会完整渲染。

---

上一页设置参考  
Vigolium 使用分层设置系统，合并来自多个源的设置。本文档涵盖配置文件格式、环境变量以及每个可配置部分。

下一页

⌘I  
website twitter github discord x linkedin

Powered by  
本文档基于 Mintlify 构建和托管，一个开发者文档平台

本页内容  
输出格式  
控制台（默认）  
JSONL  
HTML  
SQLite  
文件系统（fs）  
严重性等级  
置信度等级  
漏洞发现结构  
保存输出  
使用 `-o/--output` 标志  
管道传递 JSONL  
内容发现和爬取输出  
内容发现输出  
爬取输出  
OAST 交互  
从数据库查询结果  
列出漏洞发现  
列出流量  
读取独立导出  
将漏洞发现或记录渲染为 Markdown
文档API 参考其他CLI 参考复制页面Vigolium CLI、扫描、Agent 运行、数据导入、服务器、数据库等常见用法示例。复制页面按命令分组，精选最常用的 Vigolium 调用示例。如需查看任何命令的完整标志列表，请运行 `vigolium <command> --help`。如需在终端中查看相同示例，请运行 `vigolium --full-example`。

## 顶层命令

```bash
vigolium --help              # 所有可用命令和全局标志
vigolium <command> --help    # 特定命令的标志和帮助信息
vigolium --full-example      # 所有常见用法的精选示例
vigolium version             # 构建和版本信息
```

全局 `--soft-fail` 标志强制退出码为 0，即使命令失败（错误仍会输出到 stderr）。它可防止失败的 vigolium 调用中断包装脚本或 CI 流水线：

```bash
vigolium scan -t https://example.com --soft-fail
```

## 扫描

对一个或多个目标运行完整的本地扫描流水线。

```bash
# 单个目标
vigolium scan -t https://example.com

# 多个目标（-t 和 -T/--target-file 均可重复使用并组合）
vigolium scan -t https://example.com -t https://api.example.com
vigolium scan -T targets.txt
vigolium scan -T prod.txt -T staging.txt

# 扫描配置文件和策略
vigolium scan -t https://example.com --strategy deep
vigolium scan -t https://example.com --scanning-profile quick
vigolium scan -t https://example.com --scanning-profile full

# 阶段控制
vigolium scan -t https://example.com --only dynamic-assessment
vigolium scan -t https://example.com --skip discovery,spidering

# 模块选择
vigolium scan -t https://example.com -m xss-reflected,sqli-error
vigolium scan -t https://example.com --module-tag spring --module-tag injection

# 输出和报告
vigolium scan -t https://example.com --format jsonl -o results.jsonl
vigolium scan -t https://example.com --format html -o report.html
vigolium scan -S -t https://example.com --format sqlite -o scan      # 独立的 .sqlite 文件（仅无状态）
vigolium scan -t https://example.com --format fs -o run             # 可浏览的 run-traffic/ + run-findings/ 目录树

# 扫描后将结果打印到标准输出（与 -S --silent 配合使用；也适用于 scan-url/scan-request/run）
vigolium scan-url -S -t https://example.com --silent --print-finding        # 漏洞发现以 Markdown 格式输出
vigolium scan-url -S -t https://example.com --silent --print-traffic-tree   # 流量以主机/路径树形式输出
vigolium scan-url -S -t https://example.com --silent --print-traffic        # 原始请求/响应对

# 网络控制
vigolium scan -t https://example.com --proxy http://127.0.0.1:8080
vigolium scan -t https://example.com -c 100 --rate-limit 200
vigolium scan -t https://example.com --scanning-max-duration 2h

# CI/Agent 退出码门控
vigolium scan -t https://example.com --fail-on high       # 存在任何高严重性漏洞发现时返回非零退出码
vigolium scan -t https://example.com --fail-on high --soft-fail   # 显示错误但始终返回 0

# 自定义 JS 插件
vigolium scan -t https://example.com --ext custom-check.js
vigolium scan -t https://example.com --ext-dir ./my-extensions
vigolium scan -t https://example.com --only extension --ext custom-check.js

# 项目范围
vigolium scan -t https://example.com --project-name my-project

# OAST 和已知问题扫描
vigolium scan -t https://example.com --oast-url https://interact.sh/abc123
vigolium scan -t https://example.com --known-issue-scan-tags cve,misconfig --known-issue-scan-severities critical,high
```

## 并行与隔离扫描

同时扫描多个目标，或让多个并行扫描共享一个数据库而无需写入争用。

```bash
# 将目标列表分发到 N 个隔离的子进程，每个主机一个输出文件
vigolium scan -T targets.txt -P 4 --stateless --split-by-host --format jsonl -o results

# 并行扫描到共享数据库：每个工作进程扫描到私有临时数据库，然后合并到 --db
vigolium scan -T targets.txt -P 4 --db-isolate --db shared.db --format jsonl,html -o report

# 单次扫描到私有临时数据库，然后合并回 --db（无写入争用）
vigolium scan -t https://example.com --db-isolate --db shared.db

# 恢复中断的无状态分发扫描（跳过已完成的目标）
vigolium scan -T targets.txt -P 4 --stateless --split-by-host --format jsonl -o results --resume
vigolium scan --resume                                              # 自动发现当前目录中的 *.progress.json 文件
```

- `-P, --parallel N` — 同时扫描最多 N 个目标，作为隔离的子进程运行。需要 `-S --split-by-host`（每个主机输出）或 `--db-isolate`（合并到一个 `--db`）。实际并发请求数 ≈ N × `--concurrency`。
- `--db-isolate` — 扫描到私有临时 SQLite 数据库，最后将结果合并到 `--db`（仅 SQLite；不能与 `--stateless` 组合）。
- `--split-by-host` — 在无状态多目标模式下，为每个目标写入单独的 `base-<host>.<ext>` 输出文件。
- `--resume` — 从 `<output>.progress.json` 清单恢复之前的 `-S -T --split-by-host -P` 运行，仅扫描未完成的目标。单独运行（不带其他标志）可自动发现当前目录中的清单并重新启动保存的运行。

## 运行单个阶段

`vigolium run <phase>` 是 `scan --only <phase>` 的别名，适用于只需要流水线中特定阶段的情况。

```bash
vigolium run discover -t https://example.com
vigolium run spidering -t https://example.com
vigolium run dynamic-assessment -t https://example.com
vigolium run dynamic-assessment -t https://example.com --module-tag spring
vigolium run external-harvest -t https://example.com
vigolium run known-issue-scan -t https://example.com
vigolium run known-issue-scan -t https://example.com --known-issue-scan-tags cve --known-issue-scan-severities critical,high
vigolium run extension -t https://example.com --ext custom-check.js
vigolium run deparos -t https://example.com
vigolium run dast -t https://example.com
```

## 输入模式

从 OpenAPI、Burp、curl、HAR 或标准输入向扫描提供流量。

```bash
vigolium scan -I openapi -i openapi.yaml -t https://api.example.com
vigolium scan -I burp    -i burp-export.xml -t https://example.com
vigolium scan -I curl    -i requests.txt
vigolium scan -I har     -i traffic.har
cat urls.txt | vigolium scan -i -
```

运行 `vigolium --list-input-mode` 可查看所有支持的输入格式及示例。

## 数据导入

将 HTTP 流量推送到数据库而不运行扫描，适用于在扫描前构建项目语料库，或将流量发送到远程服务器。

```bash
vigolium ingest -t https://example.com -I openapi -i spec.yaml
vigolium ingest -t https://example.com -I burp    -i export.xml
cat urls.txt | vigolium ingest -i -

# 发送到远程 Vigolium 服务器
vigolium ingest -s http://server:9002 -i api.yaml -I openapi
```

## 服务器

启动 REST API 和数据导入代理。

```bash
vigolium server                                              # 使用配置中的默认主机/端口
vigolium server --host 0.0.0.0 --service-port 8443
vigolium server --no-auth                                    # 仅限本地使用 — 禁用 Bearer 身份验证
vigolium server -t https://example.com --scan-on-receive     # 自动扫描导入的流量（-S 是 --scan-on-receive 的简写）
vigolium server -S --passive-only                            # 仅被动模块 — 不发送主动流量；包括秘密检测
vigolium server --mirror-fs ./mirror                         # 将导入的流量和漏洞发现实时镜像到文件系统目录树

# 透明导入代理（记录流经的流量）
vigolium server --ingest-proxy-port 9003

# 通过生成的 MITM CA 拦截 HTTPS（信任启动时打印的 CA 证书）
vigolium server --ingest-proxy-port 9003 --proxy-mitm -S
vigolium server --ingest-proxy-port 9003 --proxy-mitm --proxy-insecure   # 跳过上游 TLS 验证

# 导出 MITM CA 证书并退出（如果需要则生成）
vigolium server --export-ca ./vigolium-ca.pem
```

有关完整的 MITM 工作流程，请参阅透明代理。

`--passive-only`（与 `--scan-on-receive` 一起使用）将扫描限制为仅被动模块 — 不发送主动扫描流量，并包含秘密检测。这是分析转发的 Burp/代理流量的最安全方式。与 `--full-native-scan-on-receive` 结合使用时仍会爬取（内容发现 + 爬取会发送请求）；如需零主动流量，请使用 `--scan-on-receive` 而不带 `full-native` 标志。

## 数据库与结果

浏览、导出和清理扫描数据。

```bash
# 浏览
vigolium db ls
vigolium db ls --table findings
vigolium db stats
vigolium db stats --detailed
vigolium traffic                  # `db ls --table http_records` 的别名
vigolium traffic login            # 过滤与登录相关的记录
vigolium finding                  # 模糊搜索漏洞发现
vigolium finding xss --markdown   # 将匹配结果以 Markdown 格式（证据 + HTTP 块）输出到标准输出
vigolium finding --min-severity high --confidence certain,firm   # 按最低严重性和置信度过滤

# 直接读取独立导出文件（关闭项目范围，从不写入数据库）
vigolium finding -S --db ./scan.jsonl --min-severity high
vigolium traffic -S --db ./scan.sqlite --status 500 -n 20

# 导出
vigolium export --format jsonl -o full-export.jsonl
vigolium export --format jsonl --only findings
vigolium export --format jsonl --only findings,http
vigolium export --format html -o report.html
vigolium export --format fs -o run            # 可浏览的 run-traffic/ + run-findings/ 目录树（也适用于 `db export`）

# 清理
vigolium db clean --scan-uuid my-scan
```

`finding/traffic` 接受 `-S/--stateless` + `--db <file>` 来直接读取 `--format jsonl` 导出或独立的 `.sqlite` 文件，以及 `--markdown` 将匹配项以 Markdown 格式打印（在 `-S` 下，添加 `--compact` 可窗口化长响应中的匹配部分）。

## 导入

将外部扫描数据拉回数据库。输入类型会根据路径自动检测。

```bash
# JSONL 导出（http_record + finding 信封，例如来自 `vigolium export --format jsonl`）
vigolium import full-export.jsonl

# 审计输出文件夹（包含 audit-state.json + findings-draft/）
vigolium import ./audit-output/

# 将另一个 vigolium SQLite 扫描数据库合并到默认数据库
vigolium import other-vigolium-scan.sqlite

# 合并到显式目标（--db 是目标数据库）
vigolium import --db team.sqlite other-vigolium-scan.sqlite

# 将多个独立扫描数据库合并为一个（幂等 — 重复运行无操作）
for f in scans/*.sqlite; do vigolium import --db combined.sqlite "$f"; done

# 压缩归档（.tar.gz / .tgz / .zip）或云存储对象
vigolium import ./scan-bundle.tar.gz
vigolium import gs://<project-uuid>/imports/scan.tar.gz
```

SQLite 数据库输入（通过其魔数头检测，任何扩展名均可）是无损、幂等的 SQLite→SQLite 合并：HTTP 记录、漏洞发现、扫描、Agent 扫描、OAST 交互和项目按其自然键去重，每行保留其原始项目。重新导入同一数据库不会添加任何内容。目标数据库是 `--db` 指定的数据库（如果省略 `--db`，则为配置的默认数据库）。添加 `-j/--json` 可打印每个表的合并摘要（插入行数 vs. 跳过行数）。这与 `scan -S --format sqlite` 配合使用：分发每个主机的 `.sqlite` 文件，然后合并回一个可查询的数据库。

## 重放

重新发送存储的流量 — 带或不带变异 — 以确认漏洞发现、模糊测试参数或将整个语料库推回代理。横幅被抑制，以便批量输出保持管道清洁。

```bash
# 使用 SQLi 载荷确认存储的记录（基线 vs. 变异差异）
vigolium replay --record-uuid abc12345 -m 'name=id,payload=1 OR 1=1'

# 使用 XSS 载荷重放漏洞发现的存储证据
vigolium replay --finding-id 42 -m 'name=q,payload=<svg/onload=alert(1)>'

# 逐字重放 curl 命令（通过重新发送自动建立基线）
vigolium replay -i "curl -X POST https://example.com/api/login -d 'u=admin'"

# 通过持久 cookie jar 进行多步骤身份验证
vigolium replay --session-id login -i curl-login.sh
vigolium replay --session-id login --record-uuid <action-uuid>

# 针对与基线不同的环境进行确认
vigolium replay --record-uuid abc12345 --target https://staging.example.com \
                 -m 'name=user,payload=admin' -H 'X-Forwarded-For: 127.0.0.1'

# 合并来自已保存身份验证会话的标头（来自 `vigolium auth list`）
vigolium replay --record-uuid abc12345 --target https://staging.example.com \
                 --auth-session my-session
```

## 批量重放

使用 `--all`（或任何 `--host` / `--method` / `--status` / `--path` / `--source` / `--search` / `--body`）重放每个匹配的存储记录，通过变异/差异引擎流式输出，每个记录一个 JSONL 对象，并具有每个记录的错误隔离。

```bash
# 通过 Burp 重放所有存储的流量（JSONL 输出，每次 5 个）
vigolium replay --all --proxy http://127.0.0.1:8080 -c 5

# 从独立导出重放每个记录（关闭项目范围）
vigolium replay -S --db scan.sqlite --all --proxy http://127.0.0.1:8080 -c 5

# 在所有匹配的 GET 记录中模糊测试 'id' 参数
vigolium replay --method GET --host api.example.com -m 'name=id,payload=1 OR 1=1'
```

`--all` 解除默认的 `-n/--limit` 上限（100）；改用过滤标志缩小集合。没有 `-m/--mutate` 时，每个记录按原样重新发送。

使用 `-c/--concurrency` 进行节流（默认 10）；使用 `-n/--limit` 限制集合大小（默认 100，由 `--all` 解除）。

`-S/--stateless --db <file>` 从独立的 `.sqlite`/`.jsonl` 导出中读取基线，关闭项目范围 — 它从不写入项目数据库。

## 策略与阶段

检查扫描策略预设和构成扫描的阶段。

```bash
vigolium strategy
vigolium strategy ls
vigolium phase
```

## 模块

管理主动和被动扫描器模块。

```bash
vigolium module ls
vigolium module ls xss             # 按关键字搜索
vigolium module enable xss
vigolium module disable sqli
vigolium scan -M                   # 从 scan 命令列出所有模块
```

## 技能

将二进制文件中内置的编码 Agent 技能包安装到编码 Agent 的技能目录中（以便它们始终与 CLI 版本匹配）。请参阅在 Agent 中使用 Vigolium。

```bash
vigolium skills                                   # 列出内置技能包（别名：vigolium skills list）
vigolium skills get vigolium-scanner              # 打印技能包而不安装
vigolium skills install                           # 将 vigolium-scanner 安装到当前项目的 Claude 技能目录
vigolium skills install --agent codex --scope global   # --agent claude|codex|agents, --scope project|global
vigolium skills install --all                     # 安装所有内置技能包
vigolium skills install --dir ./.claude/skills    # 覆盖目标目录
```

## 插件

运行和管理挂钩到扫描器的 JavaScript 插件。

```bash
vigolium ext ls
vigolium ext docs
vigolium ext preset
vigolium ext example                       # 打印所有示例插件（所有支持的格式）
vigolium ext example --list                # 仅目录索引（键 + 标题）
vigolium ext example js-active-insertion   # 按目录键打印单个示例
vigolium ext example --lang yaml           # 限制为一种语言
vigolium ext eval 'vigolium.log("hello")'
vigolium ext eval --ext-file script.js
```

## 扫描范围

控制哪些内容在扫描范围内。源代码通过 `vigolium agent <subcommand>`（autopilot、swarm、query、audit）上的 `--source` 标志附加到每次扫描。

```bash
vigolium scope view
vigolium scope set host.include '*.example.com'
```

## Agent（AI）

运行 Agent 和源代码审计模式。有关子命令的完整列表，请参阅 Agent 模式。

### agent query：单次提示

```bash
vigolium agent query --source ./src --prompt-template security-code-review
vigolium agent query --source ./src --prompt-template endpoint-discovery
vigolium agent query 'review this code for vulnerabilities'
vigolium agent query --agent-label code-review --prompt-file custom-prompt.md
vigolium agent --list-templates
```

### agent swarm：AI 引导的多阶段扫描

```bash
vigolium agent swarm -t https://example.com --discover
vigolium agent swarm -t https://example.com --discover --focus 'API injection'

# 技能控制（强制加载攻击向量技能，绕过规划器选择）
vigolium agent swarm -t https://example.com --discover --skill-tag xss,idor
vigolium agent swarm -t https://example.com --discover --no-skill-filter
```

### agent autopilot：自主 Agent 扫描

```bash
# 自然语言提示 — 目标、源代码和关注点自动提取
vigolium agent autopilot "scan VAmPI source at ~/src/VAmPI on localhost:3005"
vigolium agent autopilot "test auth bypass on https://app.example.com"

# 纯目标
vigolium agent autopilot -t https://example.com/api

# 源代码感知（自动先运行 vigolium-audit 以构建上下文）
vigolium agent autopilot -t https://example.com --source ./src
vigolium agent autopilot -t https://example.com --source ./src --audit=off   # 禁用 vigolium-audit

# 通过标准输入传递 curl 命令或原始 HTTP 请求
curl -s https://example.com/api/users | vigolium agent autopilot
cat request.txt | vigolium agent autopilot -t https://example.com

# 传递 curl/原始 HTTP 作为输入
vigolium agent autopilot --input "curl -X POST -H 'Content-Type: application/json' \
  -d '{\"user\":\"admin\"}' https://example.com/api/login"

# 将 Agent 的关注点集中在特定漏洞类别上
vigolium agent autopilot -t https://example.com --focus "auth bypass and IDOR"

# 技能控制 — 按名称或标签强制加载技能，或跳过预检选择
vigolium agent autopilot -t https://example.com --skill idor-blast-radius
vigolium agent autopilot -t https://example.com --skill-tag xss,idor
vigolium agent autopilot -t https://example.com --no-skill-filter

# 强度预设 — quick（CI/PR）、balanced（默认）、deep（渗透测试）
vigolium agent autopilot -t https://example.com --intensity deep
vigolium agent autopilot -t https://example.com --source ./src --intensity quick

# 缩小源代码范围
vigolium agent autopilot -t https://example.com --source ./src \
  --files "routes/api.js,controllers/auth.js" \
  --instruction "Focus on the new payment endpoint"

# PR / 差异感知扫描
vigolium agent autopilot -t https://example.com --source ./src --diff "main...feature-branch"
vigolium agent autopilot -t https://example.com --source ./src --last-commits 3

# 后端 / 浏览器 / 身份验证
vigolium agent autopilot -t https://example.com --provider anthropic-api-key
vigolium agent autopilot -t https://example.com --browser --credentials "admin/admin123"

# 限制和预览
vigolium agent autopilot -t https://example.com --intensity deep --max-duration 4h
vigolium agent autopilot -t https://example.com --intensity quick --triage
vigolium agent autopilot -t https://example.com --source ./src --dry-run
vigolium agent autopilot -t https://example.com --source ./src --show-prompt
```

### agent audit：统一源代码审计（vigolium-audit + piolium）

针对单个源代码树运行 vigolium-audit 框架和/或 piolium，在单个 AgenticScan 下，每个驱动程序会话子目录，以及后处理漏洞发现去重。vigolium-audit 是嵌入式框架名称；CLI 驱动程序值为 `audit`。

```bash
# 默认驱动程序为 "auto"：运行 audit；仅当 audit 所需的 claude/codex CLI 缺失时回退到 piolium。平衡模式。
vigolium agent audit --source .

# 无条件连续运行两个驱动程序
vigolium agent audit --driver both --source ./backend

# 单个驱动程序 — 仅 piolium（无 audit，无回退）
vigolium agent audit --driver piolium --source ./backend --mode lite
vigolium agent audit --driver audit   --source ./backend --agent claude

# 多驱动程序，深度强度，针对远程 git URL
vigolium agent audit --driver both --source git@github.com:org/repo.git --intensity deep
vigolium agent audit --driver both --source https://github.com/org/repo.git --commit-depth 0   # 完整历史

# 覆盖 piolium 部分的提供商/模型
vigolium agent audit --driver both --source ./backend \
  --pi-provider vertex-anthropic --pi-model claude-opus-4-6

# 驱动程序特定模式（piolium=longshot/smoke, audit=mock）
vigolium agent audit --driver piolium --source ./mono-repo --mode longshot \
  --plm-longshot-langs python,go --plm-longshot-limit 200
vigolium agent audit --driver audit --source ./backend --mode mock

# 限制提交历史扫描窗口（仅 piolium）
vigolium agent audit --driver piolium --source ./backend --plm-scan-since "60 days ago"
vigolium agent audit --driver piolium --source ./backend --plm-scan-limit 500

# 跳过后处理去重或预检检查
vigolium agent audit --source ./backend --no-dedup
vigolium agent audit --source ./backend --no-preflight

# 从云存储归档拉取源代码，完成后上传结果
vigolium agent audit --source gs://my-bucket/snapshots/repo.tar.gz
vigolium agent audit --source ./backend --upload-results

# 无状态一次性运行：运行到临时数据库并自动渲染自包含 HTML 报告
vigolium agent audit --source ./backend -S

# 将 HTML 报告 + 每个运行驱动程序的原始 vigolium-results/ 目录捆绑到一个文件夹中
vigolium agent audit --driver both --source ./backend -S --output-dir ./audit-bundle
```

`-S/--stateless` 将整个审计运行到临时数据库（主数据库不受影响，类似于 `scan -S`），完成后，从运行的漏洞发现中渲染自包含 HTML 报告到 `vigolium-result/vigolium-audit-report.html`（使用 `-o/--output` 覆盖，支持 `gs://` 和 `{ts}`）。`--output-dir <dir>`（仅无状态）额外将该报告和每个运行驱动程序的原始 `vigolium-results/` 目录副本捆绑到一个文件夹中 — 单个驱动程序直接放在 `<dir>/vigolium-results/` 下，多个驱动程序在 `<dir>/<driver>/` 下命名空间。`-S` 与 `--interactive` 不兼容。`--keep-raw` 在 CLI 中默认开启（保留 `<source>/vigolium-results/` 副本）；`--clean-raw` 在运行后删除该源代码副本。

### log：重放 Agent 会话

每次 `olium agent` 运行（autopilot、swarm、query、olium）都会写入一个与 Pi 兼容的 `transcript.jsonl`。将其重放为渲染的对话，或转储原始 JSONL：

```bash
vigolium log                       # 重放最近的 Agent 会话
vigolium log <session-id>          # 重放特定会话
vigolium log --raw                 # 逐字打印原始 transcript JSONL，而不是渲染的重放
```

## 设置

```bash
vigolium config ls
vigolium config clean
vigolium init                     # 使用默认值初始化 ~/.vigolium
vigolium doctor                   # 诊断配置和工具就绪状态
vigolium auth                     # 身份验证管理工具
vigolium project                  # 管理多租户项目
```

本页涵盖了最常见的调用。每个命令都支持 `--help` 以获取完整的标志参考，大多数命令接受 `vigolium --help` 显示的全局标志。

上一页API 参考所有 Vigolium API 参考页面、扫描、漏洞发现、数据导入、Agent 控制等端点的完整索引。下一页⌘I网站TwitterGitHubDiscordXLinkedIn由 Mintlify 构建和托管，一个开发者文档平台本页内容顶层命令扫描并行与隔离扫描运行单个阶段输入模式数据导入服务器数据库与结果导入重放批量重放策略与阶段模块技能插件扫描范围Agent（AI）agent query：单次提示agent swarm：AI 引导的多阶段扫描agent autopilot：自主 Agent 扫描agent audit：统一源代码审计（vigolium-audit + piolium）log：重放 Agent 会话设置
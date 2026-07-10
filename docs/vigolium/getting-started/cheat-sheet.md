文档API参考入门Vigolium 备忘录复制页面一份一页即可复制粘贴的参考，涵盖最常用的工作流程：实时流量镜像、通过 Burp 重放、被动/秘密扫描转发流量、导入外部数据、并行扇出、恢复、规范驱动扫描、单请求扫描、内容发现、按漏洞类别或技术过滤、在数据库中浏览记录的流量、使用编码代理对漏洞发现进行分类、重现漏洞利用、设置 AI 代理以及检查或编辑配置。复制页面一份快速、可复制粘贴的 Vigolium 最常用实际工作流程参考。每个块都是独立的，请根据你的环境调整主机、端口和文件路径。有关任何命令的完整说明，请通过交叉链接查看文档的其他部分。此处的每个命令都适用于开源二进制文件。`vigolium <command> -h` 会打印任何子命令的完整标志列表。

​1. 将导入的流量和漏洞发现镜像到实时文件系统树
使用 `--mirror-fs` 运行导入服务器，将每个保存的 HTTP 记录和漏洞发现写入一个扁平、可浏览的目录树（除了数据库之外），以便持久化。这与 Vigolium Burp 插件完美配合：当你浏览时，请求/响应和漏洞发现会落到磁盘上，你的编码代理可以使用简单的 `ls` / `grep` / `jq` 读取它们。
```bash
# 将导入的流量和漏洞发现镜像到实时文件系统树。
# 当记录到达时，写入 <dir>/traffic 和 <dir>/findings。
vigolium server --mirror-fs output-dir -A
```

你在 `output-dir/` 下得到的内容：
| 路径 | 内容 |
|:-----|:-----|
| `traffic/<host>/<id>.req` | 原始请求（以 `@target <scheme>://<authority>` 行开头，然后是逐字的请求，通过删除第 1 行可重放） |
| `traffic/<host>/<id>.resp.headers` | 状态行 + 响应头 |
| `traffic/<host>/<id>.resp.body` | 响应体（已解压 gzip，以便 grep 干净） |
| `traffic/<host>/index.jsonl` | 仅追加、jq 友好的 id → method/url/status/content-type 映射 |
| `findings/<host>/<id>.md` | 每个漏洞发现，交叉链接到其 `.req` 文件 |

镜像永远不会阻塞数据库保存路径（它在后台写入器上运行），并在服务器重启后按主机恢复 id 编号。配置等效项：`server.mirror_fs_path`。添加 `-S/--scan-on-receive` 以在流量到达时也进行扫描（参见第 3 节）。从代理或 shell 读取回来：
```bash
# 列出 Vigolium 为某个主机导入的所有请求
ls output-dir/traffic/example.com/

# 查找所有提及 "token" 的漏洞发现
grep -ril token output-dir/findings/

# 以 JSON 格式提取每条记录的方法/URL/状态
jq . output-dir/traffic/example.com/index.jsonl
```

​2. 通过 Burp Suite 重放所有存储的流量
通过拦截代理重新发送存储的记录，以便你可以在 Burp 中手动检查和操作它们。保持低并发（`-c`），以免压垮代理。
```bash
# 重放项目数据库中所有存储的记录，通过 Burp
vigolium replay --all --proxy http://127.0.0.1:8080 -c 5
```

直接从独立导出重放（项目范围关闭，不写入项目数据库）：
```bash
# 批量：从独立的 .sqlite 导出重放每条记录
vigolium replay -S --db scan.sqlite --all --proxy http://127.0.0.1:8080 -c 5
```

`--all` 解除默认的 `-n/--limit` 上限（100）。你可以使用 `--host`、`--method`、`--status`、`--path`、`--source`、`--search` 或 `--body` 来缩小批量集，而不是 `--all`。`-S/--stateless` 从 `.jsonl` 导出或项目范围关闭的独立 `.sqlite` 读取记录，它永远不会写入你的项目数据库。

​3. 对转发流量进行被动/秘密扫描
当你从其他来源（Burp 插件、代理或 `vigolium ingest` 客户端）将流量转发到 Vigolium 时，以扫描接收模式启动服务器。使用 `--passive-only` 仅运行被动模块，不发送主动扫描流量，并且包含秘密检测。
```bash
# 仅被动模式，持续扫描到达的转发流量。
# 不发送主动请求；运行秘密检测。
vigolium server -S --passive-only -A

# 相同，但还将所有内容镜像到磁盘供代理读取
vigolium server -S --passive-only --mirror-fs output-dir -A

# 通过透明导入代理记录并被动扫描 HTTP(S)
vigolium server -S --passive-only --ingest-proxy-port 9003 -A
```

扫描接收模式一览：
| 标志 | 行为 |
|:-----|:-----|
| `-S / --scan-on-receive` | 持续使用动态评估阶段（主动 + 被动模块）扫描新记录。 |
| `-S --passive-only` | 仅被动模块，无主动流量。包括秘密检测、安全标头、cookie 标志、信息披露等。 |
| `-S --full-native-scan-on-receive` | 对接收到的记录运行完整的本地管道（内容发现 + 爬取 + 动态评估）。 |

服务器模式始终启用所有被动模块。`--passive-only` 只是将主动模块归零，以便扫描器分析转发的请求/响应对而不发送任何新请求，非常适合扫描你在别处捕获的流量。将 Burp 插件（或 `vigolium ingest`）指向服务器的导入端点，漏洞发现会随着流量流入而出现。请参阅服务器和导入。

​4. 将外部扫描数据导入数据库
使用 `vigolium import` 将现有数据、JSONL 导出、审计输出文件夹和存档拉入数据库。它通过检查路径自动检测输入。
```bash
# JSONL 导出（http_record + finding 信封，例如来自 `vigolium export --format jsonl`）
vigolium import data.jsonl

# 审计输出文件夹（包含 audit-state.json + findings-draft/）
vigolium import ./audit-output/

# 将外部 Vigolium 扫描数据库合并到默认数据库
vigolium import other-vigolium-scan.sqlite

# 上述任一文件的压缩存档
vigolium import bundle.tar.gz      # 也支持 .tgz、.zip

# 云存储中的远程对象（下载后导入）
vigolium import gs://<project-uuid>/<key>

# 导入并一步生成品牌化 HTML 报告
vigolium import ./audit-output/ --format html -o report.html --report-title "My Report"
```

导入的漏洞发现限定于活动项目（`--project` / `VIGOLIUM_PROJECT`）。`import` 会在新的 `--db` 路径上初始化模式，因此作为针对全新数据库的第一个命令是安全的。支持的 JSONL 类型是 `http_record` 和 `finding`；其他信封类型会被计数并跳过。
​合并独立 SQLite 数据库
`vigolium import` 通过其魔术头检测 SQLite 结果数据库（适用于 `.sqlite`、`.sqlite3`、`.db` 或无扩展名），并对扫描结果表（`http_records`、`findings`、`scans`、`agentic_scans`、`oast_interactions`、`projects`）执行无损、幂等的 SQLite→SQLite 合并，基于自然键去重：
```bash
# 将外部扫描数据库合并到默认数据库
vigolium import other-vigolium-scan.sqlite

# 合并到特定的目标数据库
vigolium import other-vigolium-scan.sqlite --db team.sqlite

# 一次合并多个扫描——按位置传递它们…
vigolium import --db combined.sqlite scan-a.sqlite scan-b.sqlite scan-c.sqlite

# …或使用 --glob-db 展开通配符（幂等，因此重新运行无操作）
vigolium import --db combined.sqlite --glob-db 'scans/*.sqlite'

# --glob-db 也适用于 JSONL 导出（每次运行使用一种格式）
vigolium import --glob-db '*.jsonl'
```

如果你更愿意读取同事的导出而不将其合并到自己的数据库中，可以在项目范围关闭的情况下就地打开它。使用 `--db` 将 `-S` 指向单个文件，或使用 `--glob-db` 一次读取多个文件（它们被合并到一个一次性内存数据库中，因此 `--glob-db` 隐含 `-S`）：
```bash
# 单个独立导出
vigolium finding -S --db scan.sqlite
vigolium traffic -S --db scan.sqlite

# 跨整个导出目录（.sqlite 和/或 .jsonl）
vigolium finding --glob-db 'scans/*.sqlite'
vigolium traffic --glob-db 'scans/*.sqlite'
vigolium export --glob-db 'scans/*.sqlite' --format jsonl -o all.jsonl
```

合并的行保留其原始 `project_uuid`，因此导入的数据保持限定于其扫描时的项目。两次导入同一数据库不会添加任何内容（基于自然键去重）。需要 SQLite 目标——Postgres 目标会返回一个明确的错误。要在扫描期间将许多并行扫描合并到一个共享数据库中，请使用 `--db-isolate`（参见第 5 节）。

​5. 并行扫描大量目标列表
将 Vigolium 指向一个大型目标列表，并使用 `-P/--parallel N` 一次扫描多个主机。每个目标在其自己的隔离子进程中运行，因此工作器之间没有交叉污染，并且每个子进程保持自己的 `--concurrency`，这意味着实际在途请求大约为 N × `--concurrency`。`-P` 需要两种输出策略之一，以便结果不会冲突：
- `--stateless --split-by-host`——每个目标针对自己的临时数据库运行，并写入单独的按主机输出文件（`base-<host>.<ext>`）。不持久化任何内容。最适合无状态、即发即弃的批次。
- `--db-isolate`——每个工作器扫描到私有的临时 SQLite 数据库，然后在结束时将其结果合并到共享的 `--db`（或默认数据库）。这允许多个并行扫描共享一个数据库而无需写入争用，之后你可以从合并的数据库导出一个统一的报告。

```bash
# 无状态扇出：每个主机生成 JSONL + HTML 文件，一次 3 个目标
vigolium scan -T list-of-targets.txt -P 3 \
  --stateless --split-by-host \
  --format jsonl,html --output prefix-output \
  --fuzz-wordlist ~/Tools/contents/fast.txt

# 共享数据库扇出：一次 4 个目标合并到一个 local.db，一个统一输出
vigolium scan -T list-of-targets.txt -P 4 \
  --db-isolate --db local.db \
  --format jsonl,html --output report \
  --fuzz-wordlist ~/Tools/contents/fast.txt
```

`--db-isolate` 仅适用于 SQLite，不能与 `--stateless` 结合使用（它们是避免写入争用的两种不同方式）。在 `-P` 批次期间按 Ctrl-C 被视为操作员停止：未启动和提前结束的目标被报告为“未扫描”而不是失败。

​6. 恢复大型并行扫描
无状态并行扇出（`-S -T --split-by-host -P`）会写入一个微小的行光标清单 `<output>.progress.json`，跟踪已成功完成的目标。如果批次被中断（Ctrl-C、崩溃、CI 超时），使用 `--resume` 重新运行以跳过已完成的目标，仅扫描剩余部分。Vigolium 还会在 Ctrl-C/失败时打印一个可复制粘贴的恢复命令：
```bash
# 原始运行
vigolium scan -T targets.txt -P 4 --stateless --split-by-host --format jsonl -o results

# 仅恢复未完成的目标
vigolium scan -T targets.txt -P 4 --stateless --split-by-host --format jsonl -o results --resume
```

运行 `vigolium scan --resume` 不带其他标志，它会自动发现当前目录中的 `*.progress.json` 并从其中重新启动保存的运行（当存在多个清单时，传递 `-o <prefix>` 以消除歧义）。`--resume` 目前仅适用于并行扇出（`-S -T --split-by-host -P > 1`）。恢复普通的顺序扫描会完整重新运行。

​7. 从 OpenAPI / Swagger / Postman / Burp / HAR 输入扫描
使用 `-i <file>` 提供规范或捕获，并使用 `-I <format>` 选择格式。跳过自动发现/爬取，Vigolium 精确扫描输入中定义的端点。
```bash
# OpenAPI / Swagger 规范
vigolium scan --stateless -i api.yaml -I openapi \
  -t https://api.example.com --format jsonl -o results

# Postman 集合
vigolium scan --stateless -i collection.json -I postman \
  -t https://api.example.com --format jsonl -o results

# Burp Suite XML 导出
vigolium scan --stateless -i export.xml -I burpxml --format jsonl -o results

# HAR 捕获
vigolium scan --stateless -i traffic.har -I har --format jsonl -o results

# Nuclei JSONL
vigolium scan --stateless -i nuclei.jsonl -I nuclei --format jsonl -o results
```

格式别名：`openapi/swagger`、`postman`、`burpxml/burp/burp-xml`、`burpraw/raw`、`har/http-archive`、`nuclei/nuclei-output`。运行 `vigolium scan --list-input-mode` 获取完整列表。当规范仅包含路径时，使用 `-t/--target` 提供基础 URL。

​8. 扫描单个请求，无爬取或发现
`scan-request` 针对恰好一个原始 HTTP 请求运行扫描器模块，保留其所有参数、cookie 和标头，不进行爬取或发现。
```bash
# 从包含原始 HTTP 请求的文件
vigolium scan-request -i request.txt

# 从标准输入
printf 'GET /api/users?id=1 HTTP/1.1\r\nHost: example.com\r\n\r\n' \
  | vigolium scan-request

# 从 curl 命令（自动检测）
echo "curl -X POST -d 'user=admin' https://example.com/login" \
  | vigolium scan-request
```

当请求文件仅包含路径时覆盖主机：
```bash
vigolium scan-request -i request.txt --target https://staging.example.com
```

​从标准输入管道
`scan-url` 和 `scan-request` 都会自动检测标准输入格式：纯 URL、curl 命令或原始 HTTP 请求：
```bash
# 纯 URL
echo 'https://example.com/search?q=test' | vigolium scan-url

# Curl 命令
echo "curl -H 'Content-Type: application/json' -d '{\"id\":1}' https://example.com/api" \
  | vigolium scan-url

# 原始 HTTP 请求（保留 cookie 和 body 原样）
printf 'POST /api/login HTTP/1.1\r\nHost: example.com\r\nContent-Type: application/x-www-form-urlencoded\r\nCookie: session=abc123\r\n\r\nuser=admin&pass=secret' \
  | vigolium scan-request

# 扫描剪贴板中的内容（macOS）
pbpaste | vigolium scan-url -j
```

​9. 针对一个漏洞类别或技术
每个扫描命令（`scan`、`scan-url`、`scan-request`、`run`、`ingest`）都接受两个模块过滤器。使用 `-m/--modules` 通过模块 ID/名称的模糊匹配启用子集，或使用 `--module-tag` 按标签选择（可重复，OR 组合）。两者在省略时默认为“全部”；当同时传递两者时，结果合并（并集）。
```bash
# 仅针对单个 URL 运行 XSS 模块
vigolium scan-url -t 'https://example.com/search?q=1' -m xss

# 按标签进行技术范围扫描——仅 GraphQL、Adobe AEM 或 IIS
vigolium scan -t https://example.com --module-tag graphql
vigolium scan -t https://example.com --module-tag aem
vigolium scan -t https://example.com --module-tag iis

# 组合标签（OR）：运行每个 AEM 或 GraphQL 模块
vigolium scan -t https://example.com --module-tag aem --module-tag graphql

# 通过 ID 精确定位一个模块（可重复）
vigolium scan -t https://example.com -m graphql-scan
vigolium scan-url -t 'https://example.com/p?id=1' -m xss-light-url-params -m sqli
```

在过滤之前发现可用内容：
```bash
# 列出 id/name/description/tag 匹配某个术语的模块
vigolium module ls xss
vigolium module ls aem

# 转储可以传递给 --module-tag 的每个唯一标签
vigolium module ls --tags

# 匹配模块的完整描述和确认标准
vigolium module ls graphql -v
```

默认情况下，模块仅在目标检测到的技术栈匹配时触发（因此 GraphQL、AEM 和 IIS 模块在不相关的主机上保持休眠）。如果自动检测遗漏了栈，添加 `--no-tech-filter` 以无论如何运行所选模块（这由 `--intensity=deep` 自动启用）。`-m` 宽松匹配（`-m xss` 选择每个 XSS 模块）；传递完整 ID 如 `-m xss-light-url-params` 以精确定位一个。请参阅模块。

​10. 仅运行特定阶段（或跳过某些阶段）
使用 `--only` 运行阶段子集，或使用 `--skip` 从完整扫描中排除阶段。两者都接受逗号分隔的阶段名称和别名。
```bash
# 阶段隔离：仅运行一个阶段
vigolium scan -t https://example.com --only discovery
vigolium scan -t https://example.com --only known-issue-scan
vigolium scan -t https://example.com --only dynamic-assessment

# 跳过特定阶段（接受别名，如 kis = known-issue-scan）
vigolium scan -t https://example.com --skip discovery,spidering,kis
```

规范阶段名称及其接受的别名：
| 阶段 | 别名 |
|:-----|:-----|
| `discovery` | `deparos`、`discover` |
| `spidering` | `spitolas` |
| `known-issue-scan` | `cve`、`kis`、`known-issues` |
| `dynamic-assessment` | `audit`、`dast`、`assessment` |
| `extension` | `ext` |
| `external-harvest`、`ingestion` | — |

`dynamic-assessment` 是基于模块的漏洞扫描阶段的规范名称（以前称为 `audit`）。相同的阶段名称可作为 `vigolium run <phase>` 的参数，例如 `vigolium run cve`。

​11. 仅运行 JavaScript 插件
使用 `--ext`（可重复）加载一个或多个自定义 JS 插件，并使用 `--only extension` 将运行限制为插件阶段，不运行内置模块，仅运行你的脚本。
```bash
# 仅针对目标运行你的插件
vigolium scan -t https://example.com --only extension --ext custom-check.js

# 加载多个插件
vigolium scan -t https://example.com --only extension \
  --ext custom-check.js --ext another-check.js
```

`--ext` 是一个全局标志，因此它适用于任何扫描命令。省略 `--only extension` 以将你的插件与内置模块一起运行。请参阅编写插件以了解 `vigolium.*` API。

​12. 仅内容发现
仅运行发现/模糊测试阶段，不进行爬取，不运行漏洞模块。最直接的方法是 `vigolium run discover`，阶段运行器：它执行一个命名的阶段，仅此而已。使用 `--fuzz-wordlist` 指向一个单词列表。
```bash
# 仅针对整个主机进行发现（从单词列表暴力破解路径）
vigolium run discover -S -t https://example.com --fuzz-wordlist ~/Tools/contents/fast.txt

# 使用 URL 中的内联 FUZZ 标记模糊测试特定插入点——
# 每个单词替换 FUZZ（此处：/api/<word>/users）
vigolium run discover -S --fuzz-wordlist ~/Tools/contents/fast.txt \
  -t 'https://example.com/api/FUZZ/users'
```

FUZZ 标记可以出现在目标路径的任何位置：将其放在你想要注入单词列表的位置。没有标记时，vigolium 从目标的目录进行模糊测试（相当于追加 `/FUZZ`）。你可以在完整扫描命令上使用 `--only discovery --discover` 获得相同的仅发现行为：
```bash
# 通过 `scan` 实现相同想法：关闭除发现之外的所有阶段门
vigolium scan -t https://example.com --only discovery \
  --discover --fuzz-wordlist ~/.vigolium/wordlists/fuzz.txt

# 发现作为完整扫描的一部分（模糊测试，然后继续进入漏洞扫描）
vigolium scan -t https://example.com --discover --fuzz-wordlist ~/.vigolium/wordlists/fuzz.txt
```

`vigolium run <phase>` 隔离运行一个阶段（`discover` 是 `discovery` 的别名；参见第 10 节获取完整阶段/别名列表）。`-S/--stateless` 使其成为一次性的（不写入项目数据库）；结合 `--format fs` 或 `-o` 以持久化发现的面，供以后扫描使用。`--fuzz-wordlist` 启用即时模糊测试；FUZZ 标记固定确切的插入点。请参阅发现阶段。

​13. 添加自定义 HTTP 标头
使用 `-H/--header`（可重复）注入标头，例如身份验证令牌或 cookie。适用于 `scan`、`scan-url` 和 `scan-request`。
```bash
# 在每个请求上添加一个或多个自定义标头
vigolium scan -t https://example.com \
  -H 'Authorization: Bearer eyJhbGciOi...' \
  -H 'X-Api-Key: secret' \
  -H 'Cookie: session=abc123'
```

对于更丰富的身份验证扫描（登录流程、令牌刷新、多步骤会话），请改用身份验证会话文件，请参阅身份验证和 `vigolium auth` 命令。

​14. 控制扫描速度和持续时间
使用 `--scanning-max-duration` 限制挂钟时间，并使用 `-c/--concurrency`、`-r/--rate-limit` 和 `--max-per-host` 调整吞吐量。
```bash
# 覆盖最大扫描持续时间
vigolium scan -t https://example.com --scanning-max-duration 2h

# 调整并发和速率限制
vigolium scan -T targets.txt -c 100 --rate-limit 200 --max-per-host 5
```

| 标志 | 默认值 | 效果 |
|:-----|:-------|:-----|
| `--scanning-max-duration` | 0（使用配置） | 扫描总挂钟时间的硬上限（例如 30m、1h、2h） |
| `-c / --concurrency` | 50 | 并发扫描工作器数量 |
| `-r / --rate-limit` | 100 | 每秒最大 HTTP 请求数（全局） |
| `--max-per-host` | 50 | 任何单个主机的最大并发请求数 |

降低 `--rate-limit` 和 `--max-per-host` 以对脆弱目标保持温和；提高 `-c` 和 `--rate-limit` 以针对健壮的基础设施加快速度。在 `-P/--parallel` 下，每个子进程保持自己的 `--concurrency`，因此实际在途请求大约为 P × `--concurrency`。

​15. 检查并更改任何配置值
Vigolium 设置位于 `~/.vigolium/vigolium-configs.yaml`。使用 `vigolium config view`（`config ls` 的别名）读取它们，并使用 `vigolium config set <key> <value>` 使用点符号写入它们——无需手动编辑 YAML。
```bash
# 查看所有内容，或按子字符串/模糊键匹配过滤
vigolium config view
vigolium config view notify
vigolium config view database.sqlite

# 使用通配符模式过滤（匹配完整键或任何点段）
vigolium config view 'oast*'
vigolium config view 'kno*'          # → known_issue_scan.*

# 使用 --force / -F 显示已编辑的秘密（令牌、API 密钥）
vigolium config view notify -F
```

使用点符号键设置值——以下三种形式等效，因此你可以直接从 `config view` 输出中复制一行：
```bash
vigolium config set notify.enabled true
vigolium config set database.driver postgres
vigolium config set server.service_port 8080

# 列表值键接受逗号分隔的值
vigolium config set notify.severities high,critical

# 'key = value' 和 'key=value' 也有效（粘贴友好）
vigolium config set 'oast.server_url = your-oast-domain.com'
```

`config view` 对键进行排序，并在底部打印活动配置文件路径。敏感值显示为 `[redacted]`，除非你传递 `-F/--force`。`config set` 验证键并将其写回同一文件。使用 `vigolium config clean` 将所有内容重置为干净的默认值。请参阅配置以获取完整键参考。

​16. 设置自定义 OAST 域
带外回调检测（SSRF、盲 RCE、OOB SQLi 等）默认使用 `oast.pro` 的 interactsh 服务器。使用 `oast.server_url` / `oast.token` 配置键将 Vigolium 指向你自己的服务器：
```bash
# 将 Vigolium 指向你自己的 interactsh / OAST 服务器
vigolium config set oast.server_url your-oast-domain.com
vigolium config set oast.token your-oast-token
```

或直接在 `~/.vigolium/vigolium-configs.yaml` 中设置：
```yaml
oast:
  enabled: true
  server_url: your-oast-domain.com
  token: your-oast-token
```

配置值支持 `${VAR}` / `${VAR:-default}` 扩展，因此你可以将域和令牌保留在环境变量中，而不是写入文件：
```yaml
oast:
  server_url: ${VIGOLIUM_OAST_DOMAIN:-oast.pro}
  token: ${VIGOLIUM_OAST_TOKEN}
```

OAST 默认启用（`oast.enabled: true`）。令牌是可选的（仅需要身份验证的服务器需要）。请参阅配置 → oast 中的完整字段列表、轮询间隔、宽限期和盲 XSS 载荷源。

​17. 生成静态 HTML 报告
添加 `--format html` 和 `-o/--output` 以渲染一个自包含的 ag-grid HTML 报告。你可以在扫描期间生成，或事后从已存储或导入的数据生成。
```bash
# 在扫描期间（组合格式，例如 jsonl,html）
vigolium scan -t https://example.com --format html -o report.html

# 从数据库中已有的数据
vigolium export --format html -o report.html --report-title "Acme Q3 Scan"

# 从导入的审计 / JSONL / SQLite，一步完成
vigolium import ./audit-output/ --format html -o report.html

# 跨多个独立扫描文件的一个合并报告（无需导入）
vigolium export --glob-db 'scans/*.sqlite' --format html -o report.html
```

使用相同的标志在 `export` / `import` 上品牌化和过滤报告：
```bash
vigolium export --format html -o report.html \
  --report-title "Acme External Scan" \
  --report-target https://acme.example.com \
  --severity critical,high \
  --search sqli
```

HTML 需要 `-o/--output`。其他报告格式：`report`、`pdf`（通过 headless Chrome 渲染）和 `markdown`（别名 `md`）。输出路径接受 `gs://<project>/<key>` URL 和 `{ts}` 时间戳占位符。请参阅输出和报告 → HTML。

​18. 扫描完成时通知（Webhook）
Webhook 通知提供程序在每个扫描达到终端状态（完成或失败）时触发一次 POST，无论严重性如何，非常适合 CI 管道和仪表板。
```bash
# 每次扫描完成时触发 webhook POST
vigolium config set notify.enabled true
vigolium config set notify.provider webhook
vigolium config set notify.webhook.url https://hooks.example.com/vigolium
```

或在 `~/.vigolium/vigolium-configs.yaml` 中配置：
```yaml
notify:
  enabled: true
  provider: webhook               # webhook | telegram | discord ("" = 所有已配置的)
  webhook:
    url: https://hooks.example.com/vigolium
    authorization: "Bearer xxx"   # 可选的 Authorization 标头
    timeout_sec: 10
  severities: [high, critical]    # 按漏洞发现过滤 telegram/discord 警报
```

`webhook` 每次扫描触发一次，无论严重性如何；`telegram` 和 `discord` 则按漏洞发现触发，由 `notify.severities` 门控。Telegram/Discord 也支持环境变量回退（`TELEGRAM_BOT_TOKEN`、`TELEGRAM_CHAT_ID`、`DISCORD_WEBHOOK_URL`），因此你可以将令牌保留在配置文件之外。请参阅配置 → notify。

​19. 升级
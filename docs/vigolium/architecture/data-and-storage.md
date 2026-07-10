# Data & Persistence Architecture

> _体系结构系列：[overview](overview.md) · [native-scan](native-scan.md) · [agentic-scan](agentic-scan.md) · **data-and-storage** · [server-and-api](server-and-api.md)_

Vigolium 中的每条扫描路径——本地扫描、Agent 扫描、数据导入——最终都汇聚到同一持久化层。本文档解释数据如何**隔离**（多租户隔离）、**建模**（数据库模式）、**写入**（仓库模式和异步写入器）以及**在机器间移动**（云存储）。它是面向任务的 [projects.md](../projects.md) 和 [storage.md](../storage.md) 指南的体系结构配套文档；如需 CLI/API 操作指南，请参考后者。

---

## 1. Multi-Tenancy: the `project_uuid` spine

所有扫描数据按**项目**进行分区——项目是一个命名容器，包含 UUID、可选的配置覆盖以及可选的访问控制列表。每个项目没有独立的数据库；隔离是通过在每个主要表上设置 `project_uuid` 列实现的，每次读取时过滤，每次写入时标记。

```
内置默认值
  → ~/.vigolium/vigolium-configs.yaml          （全局配置）
    → ~/.vigolium/projects/<uuid>/config.yaml  （按项目覆盖）
      → --scanning-profile                     （扫描配置文件）
        → CLI 标志                            （最高优先级）
```

- **默认项目**——`00000000-0000-0000-0000-000000000001`，在 `vigolium init` 期间创建。当未选择任何项目时使用。
- **选择优先级**——`--project-uuid` > `--project-name` > `VIGOLIUM_PROJECT_UUID` > `VIGOLIUM_PROJECT`（遗留）> 默认。在服务器上，`X-Project-UUID` 请求头部起相同作用。
- **项目配置**是位于 `~/.vigolium/projects/<uuid>/config.yaml` 的部分 YAML 覆盖（与扫描配置文件结构相同）；仅覆盖其设置的键。
- **访问控制**——项目行上的 `allowed_emails` / `allowed_domains` 控制携带 `X-User-Email` 的服务器请求：精确邮箱列表优先，否则域名列表，否则开放；缺少邮箱头部则跳过检查；拒绝返回 `403`。`VIGOLIUM_PROJECT_READONLY=true` 禁用所有可变的 `project` CLI 子命令。

### 携带 `project_uuid` 的表

`scans` · `http_records` · `findings` · `scopes` · `source_repos` · `oast_interactions` · `scan_logs`

现有数据库原地迁移——该列以默认项目 UUID 作为默认值添加，因此多租户隔离之前的数据归属于默认项目。

---

## 2. The database backend

Vigolium 使用 **Bun ORM 之上的仓库模式**，具有两个可互换的后端：

| 后端 | 驱动 | 用途 |
|------|------|------|
| SQLite（默认） | `sqliteshim` → modernc | 单二进制、零配置、本地扫描 |
| PostgreSQL | `pgdriver` | 共享/服务器部署、并发写入者 |

模式有意设计为**反范式化**——没有独立的主机或参数表；JSONB 列承载结构化子数据。这使得单个 `http_records` 行自包含，避免了热数据导入路径上的连接扇出。

### Core models（`pkg/database/models.go`）

**`HTTPRecord`**（表 `http_records`）——数据导入流量的基本单位：

- *标识*：`UUID`（主键）、`RequestHash`（原始请求的 SHA-256，用于按源去重）
- *主机*：`Scheme`、`Hostname`、`Port`、`IP`
- *请求*：`Method`、`Path`、`URL`、`RequestHeaders`（JSONB）、`RawRequest`、`RequestBody`
- *响应*：`StatusCode`、`ResponseHeaders`（JSONB）、`RawResponse`、`ResponseBody`、`ResponseTitle`、`ResponseWords`
- *衍生*：`Parameters`（JSONB 数组，元素为 `EmbeddedParam`）、`RiskScore`、`Remarks`
- *元数据*：`Source`、`SentAt`、`ReceivedAt`、`CreatedAt`

**`Finding`**（表 `findings`）——检测到的问题：

- *标识*：`ID`（自增）、`FindingHash`（唯一约束 → 去重键，由 `ResultEvent` ID 设置）
- *模块*：`ModuleID`、`ModuleName`、`Description`、`Severity`、`Confidence`
- *证据*：`MatchedAt`（JSONB）、`ExtractedResults`、`Request`、`Response`、`AdditionalEvidence`（合并的重复请求/响应对，上限 10 条）
- *关系*：`HTTPRecordUUIDs`（JSONB）、`ScanUUID`；`finding_records` 关联表是与 HTTP 记录的多对多链接。

Agent 产生的漏洞发现（autopilot/swarm/audit/piolium）流入**相同的 `findings` 表**，通过来源标记，使本地和 AI 结果共存并一起去重。

### Converters（`pkg/database/converters.go`）

内存中的扫描类型从不直接接触数据库。`HTTPRecord.FromHttpRequestResponse()` 和 `Finding.FromResultEvent()` 是唯一的接缝——它们生成 UUID、计算哈希、解析 URL、提取标题和统计字数，将持久化关注点隔离在执行器和模块之外。

---

## 3. Write paths

| 方法（`pkg/database/repository.go`） | 角色 |
|------|------|
| `SaveRecord()` / `SaveRecordsBatch()` | 单条 vs. 批量 INSERT（批量 = 一个事务） |
| `SaveFinding()` | `INSERT … ON CONFLICT (finding_hash) DO NOTHING` + 证据追加 + 关联表行 |
| `DeduplicateFindings()` | 阶段后分组：合并共享 `(module_id, severity, matched_at URL)` 的漏洞发现 |
| `CreateScanWithCursor()` / `CountRecordsAfterCursor()` | 游标簿记，用于增量重新扫描 |
| `GetRecordsWithResponseBody()` | UUID 游标分页，用于批量扫描器（如 Kingfisher） |
| `UpdateRiskScores()` | 批量 `CASE/WHEN` UPDATE，每语句 500 个 UUID |

### Async batched ingestion — `RecordWriter`

高吞吐量数据导入（代理捕获、批量导入、爬取）**不**同步调用仓库。`pkg/database/record_writer.go` 使用缓冲通道作为前端：

```
Write() ──► 缓冲通道（容量 4096）──► 单个 flushLoop goroutine
                                            │  每批 128 条，或每 50ms 触发
                                            ▼
                                   repo.SaveRecordsBatch()  （一个事务）
```

每个调用者阻塞直到其行被刷新，并通过每个请求的结果通道获得 `WriteResult{UUID, Err}`——背压由通道容量提供，顺序得到保证，数据库看到的是大批量事务而非逐条写入。

> **SQLite DSN 说明：** modernc 驱动需要以 `_pragma=name(value)` 形式设置 pragma；`mattn` 风格的 `_busy_timeout=` 会被静默忽略。调优并发写入行为时需注意。

---

## 4. Deduplication

按设计分为两层：

1. **按源的 HTTP 记录去重**——`RequestHash`（原始请求的 SHA-256）加上 `DeduplicateRecordsBySource` 可折叠同一源内重复导入的相同请求。
2. **漏洞发现去重**——`finding_hash` 唯一约束在插入时防止精确重复；`DeduplicateFindings()` 在阶段后运行，对近似重复（相同模块/严重性/URL）进行**分组**，将额外的请求/响应对折叠到幸存者的 `AdditionalEvidence` 中（有上限）。多驱动 `audit` 命令在其所有驱动退出后，还会运行一次项目级的漏洞发现去重。

---

## 5. Cloud storage (optional)

存储**默认禁用**。启用后（`storage.enabled: true`），一个单一的 minio-go S3 客户端与 GCS（HMAC）、S3 或自托管 MinIO 通信——驱动不同，其余部分相同。

```
gs://<project-uuid>/<key>   ⇒   s3://<storage.bucket>/<project-uuid>/<key>
```

关键的体系结构要点：**项目 UUID 是桶内前缀，而非桶名**。每个键都经过验证（`storage.ValidateKey` 拒绝 `..`、反斜杠、绝对路径）并在服务器端添加项目前缀，因此一个桶可以安全地容纳多个项目，且客户端无法访问自身范围之外的数据。

| 常规前缀 | 生产者 |
|----------|--------|
| `ugc/<file>` | `vigolium storage upload`（默认） |
| `imports/<base>-<ts>.<ext>` | `vigolium import --upload` |
| `native-scans/<scan-uuid>/results.tar.gz` | `vigolium scan --upload-results` |
| `agentic-scans/<run-uuid>/results.tar.gz` | `vigolium agent … --upload-results` |

`gs://` URL 是一等输入/输出：`vigolium import gs://…` 先下载后导入（检测 `.tar.gz`/`.zip` 内的审计文件夹或 JSONL），任何 `export -o gs://…` 先本地写入，成功后上传。`{ts}` 和 `{project-uuid}` 占位符可在任何 `-o` 路径中展开。`bundle` 导出格式往返传输完整快照（JSONL + HTML 报告 + 清单 + Agent 会话目录），另一台机器可重新导入。

---

## Related

- [projects.md](../projects.md) —— 项目 CLI/API 操作指南和访问控制管理
- [storage.md](../storage.md) —— 完整的 `vigolium storage` 命令和 `gs://` 参考
- [native-scan.md](native-scan.md) —— 第 11 阶段追踪从 `ResultEvent` 到数据库行的漏洞发现流程
- [configuration.md](../configuration.md) —— `storage:` 和项目配置 YAML 块

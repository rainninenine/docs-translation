# 文档API参考本地扫描扫描剖析复制页面追踪HTTP请求在Vigolium扫描中的完整生命周期，从CLI调用到漏洞发现。复制页面本文档追踪了HTTP请求在Vigolium扫描中的完整生命周期，从命令行执行 `vigolium scan -t https://example.com` 到终端输出漏洞发现。这是一份面向贡献者的架构深度解析，旨在帮助理解端到端的扫描流水线。

## 高级流水线

```text
CLI invocation
  │
  ▼
┌──────────────────────┐
│  CLI Entry & Config  │  cmd/vigolium/main.go → pkg/cli/scan.go
│  Flag parsing, config│  Config loading, strategy/profile, DB init
│  loading, DB init    │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│   Input Parsing      │  pkg/input/source/
│   URL/file/stdin →   │  InputSource.Next() → WorkItem
│   WorkItem stream    │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  Runner Orchestration│  internal/runner/runner.go
│  6-phase pipeline:   │  Heuristics → Harvest → Discovery →
│  build infra, run    │  Spidering → KnownIssueScan →
│  phases in order     │  Dynamic-Assessment
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│     Executor         │  pkg/core/executor.go
│  Worker pool feeds   │  feedItems() → worker() → processItem()
│  items to modules    │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  Module Dispatch     │  pkg/modules/
│  Passive (sequential)│  ScanPerHost → ScanPerRequest
│  Active (parallel)   │  ScanPerHost/Request/InsertionPoint
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│   Result Emission    │  pkg/output/output.go
│  Post-hooks → DB     │  assignModuleInfo → emitResult →
│  save → output write │  SaveFinding → OnResult → Notify
└──────────────────────┘
```

## 阶段1：CLI入口与配置

### 入口点
`cmd/vigolium/main.go` 打印横幅（除非使用 `--json` 或某些子命令抑制），然后调用 `cli.Execute()`，后者调用 Cobra 根命令。

### 根命令，`pkg/cli/root.go`
`rootCmd.PersistentPreRunE` 在每个子命令之前触发，并执行以下操作：

- 通过 `initLogger()` 初始化全局 `zap.Logger`。
- 如果 `--proxy` 为空，则回退到 `VIGOLIUM_PROXY` 环境变量。
- 通过 `ensureInitialized()` 执行首次设置，创建 `~/.vigolium/` 目录，并在不存在时写入默认配置、配置文件、提示模板和技能。
- 处理提前退出的标志：`--list-modules`、`--list-input-mode`、`--full-example`。

### 扫描命令，`pkg/cli/scan.go`
`runScanCmd()` 是扫描流程的核心。它按顺序执行以下步骤：

1. 将全局标志复制到 `scanOpts`（`*types.Options`）：目标、并发数、超时、模块、代理、格式、阶段等。
2. 协调 `--json` 和 `--format`：如果设置了 `--json` 且格式仍为默认的 `"console"`，则切换为 `"jsonl"`。
3. 加载配置：`config.LoadSettings(configPath)` 读取 `~/.vigolium/vigolium-configs.yaml`。对来源模式、OAST URL 和数据库设置应用 CLI 覆盖。验证数据库、插件和策略配置。
4. 解析扫描配置文件：优先级为 `--scanning-profile` 标志 > `settings.ScanningStrategy.ScanningProfile`。配置文件从 `~/.vigolium/profiles/` 或嵌入式预设加载，并通过 `config.ApplyProfile()` 应用。
5. 解析扫描策略：优先级为 `--strategy` 标志 > `settings.ScanningStrategy.DefaultStrategy`。策略决定启用哪些阶段（发现、爬取、KnownIssueScan 等）。
6. 解析启发式检查级别：`--skip-heuristics` > `--heuristics-check` > 配置 > 默认 `"basic"`。
7. 阶段隔离：`--only` 和 `--skip` 互斥。`--only <phase>` 启用单个阶段并禁用所有其他阶段。`--skip <phase>` 禁用特定阶段。阶段别名已规范化：`deparos/discover` → `discovery`，`spitolas` → `spidering`。`dynamic-assessment` 别名作为 `audit` 的向后兼容别名被接受。
8. 验证 HTML 输出：`--format html` 需要 `--output`，且仅允许与 `--only discovery` 或 `--only spidering` 一起使用。
9. 应用扫描速率：除非在 CLI 上显式设置，否则使用配置中的并发数和每主机最大数。
10. 初始化数据库：`database.NewDB()` → `CreateSchema()` → `database.NewRepository()`。
11. 处理 `--source`：克隆 Git URL 或解析本地路径，将源代码仓库链接到数据库中的目标。
12. 分支到三个执行路径之一：

- 有 `--input` 文件？ ──是──▶ `runScanWithIngest()` 解析文件，创建 InputSource，运行
- 否
- 有目标？ ──否──▶ `runDBScan()` 扫描现有数据库记录（空来源）
- 是
- `runner.New(scanOpts)` ──────────────────▶ 基于目标：从 CLI 目标构建来源
  `.SetSettings(settings)`
  `.SetRepository(repo)`
  `.RunNativeScan()`
  `.Close()`

## 阶段2：输入解析

### InputSource 接口，`pkg/input/source/source.go`
输入源提供基于拉取的工作项流：
```go
type InputSource interface {
    Next(ctx context.Context) (*work.WorkItem, error)
    Close() error
}
```

返回约定：`(*WorkItem, nil)` = 下一个项，`(nil, io.EOF)` = 来源耗尽，`(nil, context.Canceled)` = 已取消。
可选的 `Countable` 接口添加 `Count() int64` 用于进度跟踪。

### InputSource 实现
| 类型 | 文件 | 描述 | 可计数 |
|------|------|------|--------|
| TargetSource | source.go | 遍历 CLI `-t` 目标，通过 `GetRawRequestFromURL()` 构建 GET 请求 | 是 |
| FileSource | file.go | 通过格式特定解析器解析输入文件（OpenAPI、Burp、HAR、cURL 等） | 是 |
| StdinSource | stdin.go | 从标准输入逐行读取 URL | 否 |
| SingleSource | single_source.go | 返回单个项，然后 EOF。用于 `scan-url`/`scan-request` | 是 (1) |
| MultiSource | multi.go | 按顺序依次耗尽子来源 | 是（总和） |
| ConcurrentMultiSource | concurrent.go | 并发读取所有子来源。用于基于队列的来源 | 否 |
| ExternalHarvesterInputSource | external_harvester_source.go | 运行外部收集（Wayback、CommonCrawl 等） | 否 |
| DeparosDiscoverySource | deparos_discovery.go | 对每个目标运行内容发现引擎 | 否 |

`NewInputSource(cfg SourceConfig)` 是工厂函数。根据配置字段，它创建 `TargetSource`、`FileSource` 和/或 `StdinSource`，并将多个来源包装在 `MultiSource` 中。

### 支持的输入格式
由 `file.go` 中的 `resolveFormat()` 解析：
| 格式名称 | 解析器 |
|----------|--------|
| urls, url, list | 行分隔的 URL |
| openapi, swagger | OpenAPI/Swagger 规范 |
| postman | Postman 集合 |
| curl | cURL 命令 |
| burpraw, burp-raw, raw | Burp 原始请求文件 |
| burpxml, burp-xml, burp | Burp XML 导出 |
| nuclei, nuclei-output | Nuclei JSONL 输出 |
| deparos, deparos-output | Deparos 发现输出 |

### WorkItem，`pkg/work/item.go`
```go
type WorkItem struct {
    Request       *httpmsg.HttpRequestResponse
    EnableModules []string   // 每个项的模块选择（空 = 全部）
    RecordUUID    string     // 预先存在的数据库记录 UUID（跳过存储）
    onComplete    func()     // 队列确认回调（未导出）
}
```

`Complete()` 在处理后调用，用于确认基于队列的来源。

## 阶段3：HTTP 类型

### HttpRequestResponse，`pkg/httpmsg/http_request_response.go`
贯穿整个流水线的核心数据类型。它将 HTTP 请求与可选的响应配对：
```go
type HttpRequestResponse struct {
    request  *HttpRequest   // 必需
    response *HttpResponse  // 可选，可能为 nil
}
```

关键方法：`Request()`、`Response()`、`HasResponse()`、`Service()`、`URL()`、`Target()`、`ID()`（主机:端口:方法的 FNV-1a 哈希）、`Clone()`、`WithResponse()`、`CreateInsertionPoints()`、`BuildRetryableRequest()`。

工厂函数：
- `GetRawRequestFromURL(url)`：从 URL 字符串构建最小 GET 请求（由 `TargetSource` 和 `StdinSource` 使用）
- `ParseRawRequest(raw)`：解析原始 HTTP 文本
- `FromStdRequest(req)`：转换标准库 `http.Request`

### HttpRequest，`pkg/httpmsg/http_request.go`
将原始 HTTP 请求字节存储为事实来源，并带有延迟解析的访问器：
```go
type HttpRequest struct {
    raw     []byte     // 事实来源
    service *Service   // 主机/端口/协议
    // 延迟解析缓存（由 ensureParsed() 填充）
    method, path string
    headers      []HttpHeader
    bodyOffset   int
    parsed       bool
    mu           sync.RWMutex
}
```

`ensureParsed()` 通过双重检查的读写互斥锁实现线程安全。它从原始字节中提取头部、方法、路径和主体偏移量。
不可变的构建器方法（`WithMethod()`、`WithPath()`、`WithHeader()`、`WithBody()` 等）返回带有重建原始字节的新 `*HttpRequest` 实例。`RequestOption` / `Apply()` 批量构建器模式仅对多次更改重建一次原始字节。

### HttpResponse，`pkg/httpmsg/http_response.go`
与 `HttpRequest` 相同的延迟解析模式：
```go
type HttpResponse struct {
    raw        []byte
    statusCode int
    headers    []HttpHeader
    bodyOffset int
    parsed     bool
    mu         sync.RWMutex
}
```

### Service，`pkg/httpmsg/service.go`
主机/端口/协议三元组：
```go
type Service struct {
    host     string   // 仅主机名（无端口）
    port     int
    protocol string   // "http" 或 "https"
}
```

## 阶段4：Runner 编排

### Runner，`internal/runner/runner.go`
Runner 是高级编排器。它构建共享基础设施并执行多阶段扫描流水线。
```go
type Runner struct {
    output            output.Writer
    options           *types.Options
    settings          *config.Settings
    inputSource       source.InputSource
    dedupManager      *dedup.Manager
    repository        *database.Repository
    heuristicsResults map[string]*HeuristicsResult
}
```

### `buildInfrastructure()`
在 `RunNativeScan()` 顶部调用一次。在 `phaseInfra` 容器中创建所有共享服务：
```go
type phaseInfra struct {
    svc           *services.Services
    httpRequester *http.Requester
    scopeMatcher  *config.ScopeMatcher
    hostLimiter   *hostlimit.HostRateLimiter
    notifier      *notify.Manager
    hookChain     *jsext.HookChain
    jsEngine      *jsext.Engine
    scanUUID      string
}
```

按顺序构建：
- **Notifier**：Telegram 和/或 Discord 后端（来自配置或环境变量）。
- **Services**：包装 Options、Notifier、DedupManager 和 HostErrors（无响应主机的断路器）。
- **HostRateLimiter**：每主机并发控制（CLI 默认每主机 30 并发，最多跟踪 1000 个主机，30 秒空闲驱逐）。
- **HTTP Requester**：支持重试、代理、重定向和中间件的 HTTP 客户端。
- **ScopeMatcher**：来自配置的主机/路径/状态/内容类型/主体字符串过滤。
- **JS Engine**：用于 JavaScript 插件的 Grafana Sobek 引擎，包括前置/后置钩子链。

### `RunNativeScan()`，7 阶段流水线
```text
RunNativeScan()
│
├── buildInfrastructure()
│
├─── Phase 0: Heuristics Check     [守卫: heuristicsCheck != "none"]
│    探测目标根页面，检测空白/JSON/SPA 响应。
│    标记要跳过爬取的目标。
│
├─── Phase 1: External Harvest     [守卫: ExternalHarvestEnabled]
│    查询 Wayback、CommonCrawl、AlienVault、URLScan、VirusTotal。
│    将发现的 URL 导入数据库（无模块，纯导入）。
│
├─── Phase 2: Discovery            [守卫: !SkipIngestion]
│    内容发现（通过 deparos 引擎暴力破解目录/文件）
│    + CLI 输入源。两者都包装在 MultiSource 中。
│    导入数据库（无模块，纯导入）。
│    回退：如果跳过导入但需要 KnownIssueScan/DA 记录，则 seedCLITargets()。
│
├─── Phase 3: Spidering            [守卫: SpideringEnabled]
│    基于浏览器的爬取（Chromium）。应用启发式过滤器。
│    通过 repository 将发现的页面存储在数据库中。
│
├─── Phase 4: KnownIssueScan       [守卫: KnownIssueScanEnabled]
│    对存储的响应主体运行 Nuclei 模板扫描 + Kingfisher 秘密检测。
│    目标通过发现的路径丰富（enrich_targets）。
│    过滤掉 secret_detect 被动模块以避免 DA 阶段重复。
│    阶段后：DeduplicateFindings() 对相同模块/URL 的发现进行分组。
│
└─── Phase 5: Dynamic-Assessment   [守卫: !SkipAudit]
     核心扫描阶段。从数据库读取记录，调度主动 + 被动模块。
     每模块发现上限抑制嘈杂模块。
     反馈循环（最多 3 轮）重新扫描新发现的 URL。
     阶段后：DeduplicateFindings() 合并冗余发现。
```

阶段 0-4 用 HTTP 记录填充数据库。阶段 5 读取这些记录并针对它们运行完整的模块流水线。

### 阶段 5 详情：KnownIssueScan
- 从数据库通过 `GetDistinctPaths()` 查询不同路径。
- 构建目标 URL，可以是路径丰富的（默认，`enrich_targets: true`）或仅主机级别。
- 针对目标运行 Nuclei 模板 + Kingfisher 秘密扫描。
- 阶段后去重：调用 `DeduplicateFindings()` 对具有相同 `(module_id, severity, matched_at URL)` 的发现进行分组。

### 阶段 6 详情：审计
- 创建带有游标跟踪的 `database.Scan` 记录。
- 从配置解析 DA 并发数（与发现并发数分开）。
- 可选启动 OAST（带外）服务。
- 运行反馈循环（最多 `maxFeedbackRounds = 3`）：
  - 创建 `OneShotDBInputSource`，读取扫描游标之后的记录。
  - 使用所有主动 + 被动模块构建 Executor，`SkipBaseline: true`（响应已在数据库中）。
  - Executor 强制执行每模块发现上限（`MaxFindingsPerModule`，默认 10），一旦模块发出这么多发现，该模块的进一步结果将被抑制。
  - 每轮后，检查是否有新创建的记录。如果没有，则提前退出。
- 阶段后去重：调用 `DeduplicateFindings()` 合并同一模块在同一 URL 上使用不同载荷触发的发现。
- 将扫描标记为完成。

## 阶段5：Executor

### Executor 结构体，`pkg/core/executor.go`
Executor 是中央调度引擎。它接收工作项，将它们分发给工作池，并调度模块。
```go
type Executor struct {
    cfg            ExecutorConfig
    source         source.InputSource
    activeModules  []modules.ActiveModule
    passiveModules []modules.PassiveModule
    httpClient     *http.Requester
    scanCtx        *modules.ScanContext
    hooks          HookRunner

    // 初始化时按扫描范围预分组
    perHostActive     []modules.ActiveModule
    perRequestActive  []modules.ActiveModule
    perIPActive       []modules.ActiveModule
    perHostPassive    []modules.PassiveModule
    perRequestPassive []modules.PassiveModule

    ipCache      *lru.Cache[string, []httpmsg.InsertionPoint]  // 4096 条目 LRU
    requestUUIDs *shardedMap   // 请求哈希 → 数据库记录 UUID
}
```

### 模块预分组
在构造时，`NewExecutor()` 将所有模块按其 `ScanScope` 位掩码预分组为五个切片。声明 `ScanScopeInsertionPoint | ScanScopeRequest` 的模块同时出现在 `perIPActive` 和 `perRequestActive` 中。这避免了每个项的 scope-check 迭代。

### `Execute()`，工作池
```go
func (e *Executor) Execute(ctx context.Context) (bool, error)
```

- 生成 `Workers` 个 goroutine，从缓冲通道读取（容量 = `Workers * 2`）。
- 在调用 goroutine 上调用 `feedItems()`（生产者循环）。
- 关闭通道，等待所有 worker 耗尽。
- 刷新被动模块（`Flusher` 接口）和 OAST 服务。
- 返回 `(foundResults, nil)`。

### `feedItems()`，生产者
对于来自 `source.Next()` 的每个项：
- **静态文件过滤器**：如果路径匹配静态文件扩展名（`.jpg`、`.css` 等），则跳过。
- **预请求范围检查**：`ScopeMatcher.InScopeRequest(host, path, "", "")`，仅主机 + 路径，无 HTTP 往返。提前拒绝明显超出范围的项。
- **主机错误检查**：如果 `HostErrors.Check(hostID)` 返回 true（主机已被断路器断开），则跳过。
- 将项发送到工作通道。

### `worker()`，消费者
每个 worker goroutine 在通道上循环：
```go
for item := range itemCh {
    e.processItem(item)
    item.Complete()
    e.statsTracker.Increment()
}
```

## 阶段6：处理项

`processItem()` 是每个项的热路径。每个通过 `feedItems()` 的项都经过以下步骤：

### 步骤1：基线 HTTP 获取
```go
if SkipBaseline && response already attached:
    use existing response (DB-sourced items in audit phase)
else:
    httpClient.Execute(request) → response
    copy response bytes from pool before Close()
    attach response to request via WithResponse()
```

响应字节从 `sync.Pool` 的回收缓冲区复制（初始 32 KiB，池返回最大 1 MiB），以减少 GC 压力。

### 步骤2：流量回调
如果配置了，调用 `OnTraffic(method, url, statusCode, contentType)`，这是一个观察者钩子，用于将流量行打印到 stderr。

### 步骤3：前置钩子
```go
hooks.RunPreHooks(request)
  → error: log and skip item
  → nil return: hook filtered it out, skip item
  → modified request: continue with transformed request
```

前置钩子可以注入认证头部、转换请求或指示完全跳过。

### 步骤4：主体大小强制执行
如果设置了 `ScopeMatcher`，检查请求和响应主体大小：
- `BodySizeDrop` → 完全丢弃项。
- `BodySizeTruncate` → 将主体截断到限制，继续扫描。
- `BodySizeSkipScan` → 截断，保存到数据库，但跳过扫描。

### 步骤5：范围检查 + 数据库保存
```go
if ScopeMatcher configured:
    check full scope (host, path, status, content types, body strings)
    if out-of-scope and ScopeOnIngest: drop entirely (no save, no scan)
    save to database
    if out-of-scope: saved but not scanned → return
else:
    save to database and continue
```

`saveToDatabase()` 调用 `repo.SaveRecord()` 并将返回的 UUID 存储在 `requestUUIDs` 分片映射中（键为请求 SHA-256 哈希），用于后续发现链接。

### 步骤6：资格预计算
`computeEligibility()` 每个项运行一次（不是每个模块）：
- 请求 nil 检查
- URL 解析检查
- 媒体/JS URL 检查（`utils.IsMediaAndJSURL`）
- HTTP 方法检查（跳过 OPTIONS、CONNECT、HEAD、TRACE）

缓存的 `baseEligible` 结果允许 executor 在基础检查会拒绝时跳过调用嵌入标准基础检查的模块的 `CanProcess()`。

### 步骤7：模块过滤器
如果 `item.EnableModules` 非空，构建基于映射的 O(1) 过滤器。否则使用 `allModulesFilter` 哨兵。

### 步骤8：被动模块执行（顺序）
```go
runPassivePerHost(request, filter)      sequential loop over perHostPassive
runPassivePerRequest(request, filter)   sequential loop over perRequestPassive
```

对于每个模块：检查过滤器 → 检查 `CanProcess()` → 调用扫描方法 → 处理结果。无 goroutine，被动模块不执行网络 I/O。

### 步骤9：主动模块执行（并行）
三个类别通过 `conc.WaitGroup` 并行运行：
```go
var g conc.WaitGroup
g.Go(func() { runActivePerHost(request, filter, eligibility) })
g.Go(func() { runActivePerRequest(request, filter, eligibility) })
g.Go(func() { runActivePerInsertionPoint(request, filter, eligibility) })
g.Wait()
```

在每个类别中，符合条件的模块也并发运行（内部 `conc.WaitGroup`）。
对于插入点类别，插入点串行迭代（一次一个），但给定点的所有符合条件的模块并发运行：
```go
insertion points = ipCache.GetOrCompute(requestHash)
for each insertionPoint:
    for each eligible module (parallel):
        module.ScanPerInsertionPoint(request, insertionPoint, httpClient, scanCtx)
```

### 并发模型总结
```text
Execute()
├── feedItems()                          [调用 goroutine，生产者]
└── Workers goroutines                   [消费者池]
    └── processItem()
        ├── Passive modules              [worker goroutine 上顺序]
        │   ├── runPassivePerHost
        │   └── runPassivePerRequest
        └── Active modules               [通过 conc.WaitGroup 3 路并行]
            ├── runActivePerHost          [内部并行：所有模块]
            ├── runActivePerRequest       [内部并行：所有模块]
            └── runActivePerInsertionPoint
                └── for each IP (串行)    [内部并行：所有模块]
```

## 阶段7：插入点

### InsertionPoint 接口，`pkg/httpmsg/insertion_point.go`
```go
type InsertionPoint interface {
    Name() string                        // 参数名称（例如 "id"、"username"）
    BaseValue() string                   // 此位置的原始值
    Type() InsertionPointType            // 一个 INS_* 常量
    BuildRequest(payload []byte) []byte  // 注入载荷后的新请求字节
    PayloadOffsets(payload []byte) []int  // 构建请求中的 [startOffset, endOffset]
}
```

### InsertionPointType 常量
| 常量 | 值 | 描述 |
|------|-----|------|
| INS_PARAM_URL | 0 | URL 查询参数值 |
| INS_PARAM_BODY | 1 | POST 主体参数值 |
| INS_PARAM_COOKIE | 2 | Cookie 值 |
| INS_PARAM_XML | 3 | XML 元素值 |
| INS_PARAM_XML_ATTR | 4 | XML 属性值 |
| INS_PARAM_MULTIPART_ATTR | 5 | 多部分属性值 |
| INS_PARAM_JSON | 6 | JSON 值 |
| INS_PARAM_AMF | 7 | AMF 参数值 |
| INS_HEADER | 32 | HTTP 头部值 |
| INS_URL_PATH_FOLDER | 33 | REST URL 路径文件夹 |
| INS_PARAM_NAME_URL | 34 | URL 参数名称 |
| INS_PARAM_NAME_BODY | 35 | 主体参数名称 |
| INS_ENTIRE_BODY | 36 | 整个请求主体 |
| INS_URL_PATH_FILENAME | 37 | REST URL 路径文件名 |
| INS_USER_PROVIDED | 64 | 用户定义位置 |
| INS_EXTENSION_PROVIDED | 65 | 插件提供位置 |
| INS_UNKNOWN | 127 | 未知/未分类 |

### InsertionPoint 实现
| 类型 | 描述 |
|------|------|
| ParameterInsertionPoint | 标准参数替换。使用基于偏移的拼接，带有类型感知的载荷编码（URL 参数/主体/cookie 使用 URL 编码，JSON 参数使用 JSON 感知，XML 使用原始）。 |
| HeaderInsertionPoint | 头部值替换。使用 `AddOrReplaceHeader()` 而不是偏移拼接。为现有的可注入头部 + 合成头部（X-Forwarded-For、X-Forwarded-Host、Referer、True-Client-IP、X-Real-IP）创建。 |
| NestedInsertionPoint | 多层编码链（例如，主体参数内 URL 编码的 JSON）。`BuildRequest()` 从内到外应用：子构建首先，然后父级对结果进行编码。 |
| EncodedInsertionPoint | 自定义编码器链。应用前缀 + 载荷 → `encoder.Encode()` → 拼接。用于复杂编码场景。 |

### LRU 缓存
Executor 维护一个 4096 条目的 LRU 缓存（`ipCache`），键为请求 SHA-256 哈希。`CreateAllInsertionPoints()` 对每个唯一请求调用一次，结果被所有扫描该请求的模块重用。

### 共享基础请求
`CreateAllInsertionPoints()` 创建单个共享的 `baseRequest` 原始字节克隆，该克隆在该调用创建的所有 `ParameterInsertionPoint` 实例之间共享。这是安全的，因为 `BuildRequest()` 从不修改共享字节，它总是分配一个新的结果切片。

## 阶段8：模块调度

### 模块接口层次结构，`pkg/modules/`
```text
Module (base)
├── ActiveModule
│   ├── ScanPerInsertionPoint(request, insertionPoint, httpClient, scanCtx)
│   ├── ScanPerRequest(request, httpClient, scanCtx)
│   ├── ScanPerHost(request, httpClient, scanCtx)
│   └── AllowedInsertionPointTypes() InsertionPointTypeSet
│
└── PassiveModule
    ├── ScanPerRequest(request, scanCtx)
    ├── ScanPerHost(request, scanCtx)
    ├── Scope() PassiveScanScope
    └── (optional) Flusher: Flush(scanCtx)
```

### ScanScope 位掩码，`pkg/modules/modkit/types.go`
```go
const (
    ScanScopeInsertionPoint ScanScope = 1 << iota  // = 1
    ScanScopeRequest                                // = 2
    ScanScopeHost                                   // = 4
)
```

模块通过 OR 常量声明一个或多个范围。Executor 在启动时使用 `ScanScopes().Has(scope)` 预分组模块。

### InsertionPointTypeSet，`pkg/modules/modkit/types.go`
一个 `uint32` 位掩码，其中每个位对应一个 `InsertionPointType`。在调用 `ScanPerInsertionPoint()` 之前由 Executor 检查：
```go
module.AllowedInsertionPointTypes().Contains(ip.Type())
```

预构建的预设：`URLParamTypes`、`BodyParamTypes`、`CookieTypes`、`HeaderTypes`、`AllParamTypes`。

### CanProcess 语义
- **主动模块**（通过 `BaseActiveModule`）：拒绝 nil 请求、无法解析的 URL、媒体/JS URL 和不可测试的 HTTP 方法（OPTIONS、CONNECT、HEAD、TRACE）。Executor 在 `computeEligibility()` 中预计算这些检查，并在基础会拒绝时跳过调用 `CanProcess()`。
- **被动模块**（通过 `BasePassiveModule`）：仅检查所需的 HTTP 事务部分（请求和/或响应）是否存在。它们处理所有内容类型，包括媒体，无方法过滤。

### 执行模式
```text
Per item:
  1. Passive per-host   → sequential loop, no goroutines
  2. Passive per-request → sequential loop, no goroutines
  3. Active per-host     → parallel: all eligible modules concurrently
  4. Active per-request  → parallel: all eligible modules concurrently
  5. Active per-IP       → for each insertion point (serial):
                             all eligible modules concurrently
```

步骤 3-5 通过 `conc.WaitGroup` 作为三个并发 goroutine 组运行。

### ScanContext，`pkg/modules/modkit/context.go`
扫描期间所有模块可用的共享资源：
```go
type ScanContext struct {
    DedupManager        *dedup.Manager
    RiskScoreUpdater    RiskScoreUpdater
    RequestUUIDResolver RequestUUIDResolver
    OASTProvider        OASTProvider
    MutationGen         MutationGenerator
    baselineCache       sync.Map  // "METHOD:host/path" → *BaselineEntry
}
```

- **DedupManager**：请求级去重。
- **OASTProvider**：生成带外回调 URL。
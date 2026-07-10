# Native Scan Architecture（本地扫描体系结构）——扫描剖析

> _体系结构系列：[overview](overview.md) · **native-scan** · [agentic-scan](agentic-scan.md) · [data-and-storage](data-and-storage.md) · [server-and-api](server-and-api.md)_

本文档追踪一个 HTTP 请求在 Vigolium 扫描中的完整生命周期——从命令行执行 `vigolium scan -t https://example.com` 到将漏洞发现写入终端。这是一份面向贡献者的体系结构深度解析，旨在帮助理解扫描管道的端到端流程。

## 高层管道

```
CLI 调用
  │
  ▼
┌─────────────────────┐
│  CLI 入口与配置      │  cmd/vigolium/main.go → pkg/cli/scan.go
│  标志解析、配置      │  配置加载、策略/配置文件、数据库初始化
│  加载、数据库初始化  │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   输入解析           │  pkg/input/source/
│   URL/文件/stdin →   │  InputSource.Next() → WorkItem
│   WorkItem 流       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  运行器编排          │  internal/runner/runner.go
│  6 阶段管道：        │  启发式 → 收集 → 爬取 →
│  构建基础设施，      │  发现 → KnownIssueScan →
│  按顺序执行阶段      │  动态评估
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│     执行器           │  pkg/core/executor.go
│  工作池将项目        │  feedItems() → worker() → processItem()
│  分发给模块          │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  模块分发            │  pkg/modules/
│  被动（顺序）        │  ScanPerHost → ScanPerRequest
│  主动（并行）        │  ScanPerHost/Request/InsertionPoint
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   结果输出           │  pkg/output/output.go
│  后置钩子 → 数据库   │  assignModuleInfo → emitResult →
│  保存 → 输出写入     │  SaveFinding → OnResult → Notify
└─────────────────────┘
```

## 阶段 1：CLI 入口和配置

### 入口点

`cmd/vigolium/main.go` 打印横幅（除非 `--json` 或某些子命令抑制它），然后调用 `cli.Execute()`，后者调用 Cobra 根命令。

### 根命令——`pkg/cli/root.go`

`rootCmd.PersistentPreRunE` 在每个子命令之前触发，并执行以下操作：

1. 通过 `initLogger()` 初始化全局 `zap.Logger`。
2. 如果 `--proxy` 为空，回退到 `VIGOLIUM_PROXY` 环境变量。
3. 通过 `ensureInitialized()` 运行首次设置——如果不存在则创建 `~/.vigolium/` 并写入默认配置、配置文件和提示模板。
4. 处理提前退出的标志：`--list-modules`、`--list-input-mode`、`--full-example`。

### 扫描命令——`pkg/cli/scan.go`

`runScanCmd()` 是扫描流程的核心。它按顺序执行以下步骤：

1. **复制全局标志**到 `scanOpts`（`*types.Options`）：目标、并发数、超时、模块、代理、格式、阶段等。
2. **协调 `--json` 和 `--format`**：如果设置了 `--json` 且格式仍为默认的 `"console"`，则切换到 `"jsonl"`。
3. **加载配置**：`config.LoadSettings(configPath)` 读取 `~/.vigolium/vigolium-configs.yaml`。对来源模式、OAST URL 和数据库设置应用 CLI 覆盖。验证数据库、插件和策略配置。
4. **解析扫描配置文件**：优先级为 `--scanning-profile` 标志 > `settings.ScanningStrategy.ScanningProfile`。配置文件从 `~/.vigolium/profiles/` 或嵌入式预设加载，通过 `config.ApplyProfile()` 应用。
5. **解析扫描策略**：优先级为 `--strategy` 标志 > `settings.ScanningStrategy.DefaultStrategy`。策略决定哪些阶段启用（发现、爬取、KnownIssueScan 等）。
6. **解析启发式检查级别**：`--skip-heuristics` > `--heuristics-check` > 配置 > 默认 `"basic"`。
7. **阶段隔离**：`--only` 和 `--skip` 互斥。`--only <phase>` 启用单个阶段并禁用所有其他阶段。`--skip <phase>` 禁用特定阶段。阶段别名被标准化：`deparos`/`discover` → `discovery`，`spitolas` → `spidering`，`ext` → `extension`。
8. **验证 HTML 输出**：`--format html` 需要 `--output`，且仅允许与 `--only discovery` 或 `--only spidering` 一起使用。
9. **应用扫描节奏**：来自配置的并发数和每主机最大数被应用，除非在 CLI 上显式设置。
10. **初始化数据库**：`database.NewDB()` → `CreateSchema()` → `database.NewRepository()`。
11. **处理 `--source`**：克隆 git URL 或解析本地路径，将源代码仓库链接到数据库中的目标。
12. **分支到三个执行路径之一**：

```
有 --input 文件？  ──是──▶  runScanWithIngest()    解析文件，创建 InputSource，运行
       │ 否
       ▼
有目标？           ──否──▶  runDBScan()            扫描现有数据库记录（空来源）
       │ 是
       ▼
runner.New(scanOpts)         ──────────────────▶    基于目标：从 CLI 目标构建来源
  .SetSettings(settings)
  .SetRepository(repo)
  .RunNativeScan()
  .Close()
```

## 阶段 2：输入解析

### InputSource 接口——`pkg/input/source/source.go`

输入来源提供基于拉取的工作项流：

```go
type InputSource interface {
    Next(ctx context.Context) (*work.WorkItem, error)
    Close() error
}
```

返回约定：`(*WorkItem, nil)` = 下一个项目，`(nil, io.EOF)` = 来源耗尽，`(nil, context.Canceled)` = 已取消。

可选的 `Countable` 接口添加 `Count() int64` 用于进度跟踪。

### InputSource 实现

| 类型 | 文件 | 描述 | Countable |
|---|---|---|---|
| `TargetSource` | `source.go` | 迭代 CLI `-t` 目标，通过 `GetRawRequestFromURL()` 构建 GET 请求 | 是 |
| `FileSource` | `file.go` | 通过格式特定解析器解析输入文件（OpenAPI、Burp、HAR、cURL 等） | 是 |
| `StdinSource` | `stdin.go` | 从 stdin 逐行读取 URL | 否 |
| `SingleSource` | `single_source.go` | 返回单个项目，然后返回 EOF。由 `scan-url`/`scan-request` 使用 | 是（1） |
| `MultiSource` | `multi.go` | 按顺序依次耗尽子来源 | 是（总和） |
| `ConcurrentMultiSource` | `concurrent.go` | 并发读取所有子来源。用于基于队列的来源 | 否 |
| `ExternalHarvesterInputSource` | `external_harvester_source.go` | 运行外部收集（Wayback、CommonCrawl 等） | 否 |
| `DeparosDiscoverySource` | `deparos_discovery.go` | 按目标运行内容发现引擎 | 否 |

`NewInputSource(cfg SourceConfig)` 是工厂函数。根据配置字段，它创建 `TargetSource`、`FileSource` 和/或 `StdinSource`，将多个来源包装在 `MultiSource` 中。

### 支持的输入格式

由 `file.go` 中的 `resolveFormat()` 解析：

| 格式名称 | 解析器 |
|---|---|
| `urls`、`url`、`list` | 换行分隔的 URL |
| `openapi`、`swagger` | OpenAPI/Swagger 规范 |
| `postman` | Postman 集合 |
| `curl` | cURL 命令 |
| `burpraw`、`burp-raw`、`raw` | Burp 原始请求文件 |
| `burpxml`、`burp-xml`、`burp` | Burp XML 导出 |
| `nuclei`、`nuclei-output` | Nuclei JSONL 输出 |
| `deparos`、`deparos-output` | Deparos 发现输出 |

### WorkItem——`pkg/work/item.go`

```go
type WorkItem struct {
    Request       *httpmsg.HttpRequestResponse
    EnableModules []string   // 每个项目的模块选择（空 = 全部）
    RecordUUID    string     // 预先存在的数据库记录 UUID（跳过存储）
    onComplete    func()     // 队列确认回调（未导出）
}
```

`Complete()` 在处理后调用，用于确认基于队列的来源。

## 阶段 3：HTTP 类型

### HttpRequestResponse——`pkg/httpmsg/http_request_response.go`

贯穿整个管道的核心数据类型。它将 HTTP 请求与可选的响应配对：

```go
type HttpRequestResponse struct {
    request  *HttpRequest   // 必需
    response *HttpResponse  // 可选，可能为 nil
}
```

关键方法：`Request()`、`Response()`、`HasResponse()`、`Service()`、`URL()`、`Target()`、`ID()`（`host:port:method` 的 FNV-1a 哈希）、`Clone()`、`WithResponse()`、`CreateInsertionPoints()`、`BuildRetryableRequest()`。

工厂函数：
- `GetRawRequestFromURL(url)` — 从 URL 字符串构建最小 GET 请求（由 `TargetSource` 和 `StdinSource` 使用）
- `ParseRawRequest(raw)` — 解析原始 HTTP 文本
- `FromStdRequest(req)` — 转换标准库 `http.Request`

### HttpRequest——`pkg/httpmsg/http_request.go`

以原始 HTTP 请求字节作为数据源存储，带有懒解析访问器：

```go
type HttpRequest struct {
    raw     []byte     // 数据源
    service *Service   // host/port/protocol
    // 懒解析缓存（由 ensureParsed() 填充）
    method, path string
    headers      []HttpHeader
    bodyOffset   int
    parsed       bool
    mu           sync.RWMutex
}
```

`ensureParsed()` 通过双重检查的读写互斥锁实现线程安全。它从原始字节中提取标头、方法、路径和正文偏移量。

不可变构建器方法（`WithMethod()`、`WithPath()`、`WithHeader()`、`WithBody()` 等）返回带有重建原始字节的新 `*HttpRequest` 实例。`RequestOption` / `Apply()` 批量构建器模式在多次更改时仅重建一次原始字节。

### HttpResponse——`pkg/httpmsg/http_response.go`

与 `HttpRequest` 相同的懒解析模式：

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

### Service——`pkg/httpmsg/service.go`

主机/端口/协议三元组：

```go
type Service struct {
    host     string   // 仅主机名（无端口）
    port     int
    protocol string   // "http" 或 "https"
}
```

## 阶段 4：运行器编排

### 运行器——`internal/runner/runner.go`

运行器是高级编排器。它构建共享基础设施并执行多阶段扫描管道。

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

### buildInfrastructure()

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
1. **通知器**——Telegram 和/或 Discord 后端（来自配置或环境变量）。
2. **服务**——封装 Options、Notifier、DedupManager 和 HostErrors（无响应主机的断路器）。
3. **HostRateLimiter**——每主机并发控制（默认：每主机 2 个并发，最多 1000 个跟踪主机，30 秒空闲驱逐）。
4. **HTTP Requester**——支持重试、代理、重定向和中间件的 HTTP 客户端。
5. **ScopeMatcher**——来自配置的主机/路径/状态/内容类型/正文字符串过滤。
6. **JS 引擎**——用于 JavaScript 插件的 Grafana Sobek 引擎，包括前置/后置钩子链。

### RunNativeScan()——6 阶段管道

```
RunNativeScan()
│
├── buildInfrastructure()
│
├─── 阶段 0：启发式检查     [守卫：heuristicsCheck != "none"]
│    探测目标根页面，检测空白/JSON/SPA 响应。
│    标记要跳过爬取的目标。
│
├─── 阶段 1：外部收集       [守卫：ExternalHarvestEnabled]
│    查询 Wayback、CommonCrawl、AlienVault、URLScan、VirusTotal。
│    将发现的 URL 导入数据库（无模块，纯导入）。
│
├─── 阶段 2：爬取           [守卫：SpideringEnabled]
│    基于浏览器的爬取（Chromium）。应用启发式过滤器。
│    通过 repository 将发现的页面存储在数据库中。
│
├─── 阶段 3：发现           [守卫：!SkipIngestion]
│    内容发现（通过 deparos 引擎暴力破解目录/文件）
│    + CLI 输入来源。两者都包装在 MultiSource 中。
│    导入到数据库（无模块，纯导入）。
│    回退：如果跳过导入但 KnownIssueScan/DA 需要记录，则 seedCLITargets()。
│
├─── 阶段 4：KnownIssueScan [守卫：KnownIssueScanEnabled]
│    在存储的响应体上运行 Nuclei 模板扫描 + Kingfisher 密钥检测。
│    目标通过发现的路径丰富（enrich_targets）。
│    过滤掉 secret_detect 被动模块以避免在 DA 阶段重复。
│    阶段后：DeduplicateFindings() 对相同 module/URL 的漏洞发现进行分组。
│
└─── 阶段 5：动态评估       [守卫：!SkipDynamicAssessment]
     核心扫描阶段。从数据库读取记录，分发
     主动 + 被动模块。每模块漏洞发现上限抑制
     噪音模块。反馈循环（最多 3 轮）重新扫描
     新发现的 URL。
     阶段后：DeduplicateFindings() 合并冗余的漏洞发现。
```

阶段 0-4 用 HTTP 记录填充数据库。阶段 5 读取这些记录并对其运行完整的模块管道。

### 阶段 4 详情：KnownIssueScan

1. 通过 `GetDistinctPaths()` 从数据库查询不同的路径。
2. 构建目标 URL——路径丰富（默认，`enrich_targets: true`）或仅主机级别。
3. 对目标运行 Nuclei 模板 + Kingfisher 密钥扫描。
4. **阶段后去重**：调用 `DeduplicateFindings()` 对具有相同 `(module_id, severity, matched_at URL)` 的漏洞发现进行分组。

### 阶段 5 详情：动态评估

1. 创建带游标跟踪的 `database.Scan` 记录。
2. 从配置解析 DA 并发数（与发现并发数分开）。
3. 可选启动 OAST（带外）服务。
4. 运行**反馈循环**（最多 `maxFeedbackRounds = 3`）：
   - 创建一个 `OneShotDBInputSource`，读取扫描游标之后的记录。
   - 构建一个包含所有主动 + 被动模块的执行器，`SkipBaseline: true`（响应已在数据库中）。
   - 执行器强制执行每模块漏洞发现上限（`MaxFindingsPerModule`，默认 10）——一旦模块发出这么多漏洞发现，该模块的进一步结果被抑制。
   - 每轮之后，检查是否有新创建的记录。如果没有则提前退出。
5. **阶段后去重**：调用 `DeduplicateFindings()` 合并同一模块使用不同载荷在同一 URL 上触发的漏洞发现。
6. 将扫描标记为已完成。

## 阶段 5：执行器

### 执行器结构——`pkg/core/executor.go`

执行器是中央分发引擎。它接收工作项，将其分发给工作池，并分发模块。

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

在构造时，`NewExecutor()` 将所有模块按其 `ScanScope` 位掩码预分组到五个切片中。声明 `ScanScopeInsertionPoint | ScanScopeRequest` 的模块同时出现在 `perIPActive` 和 `perRequestActive` 中。这避免了每次检查作用域的迭代。

### Execute()——工作池

```go
func (e *Executor) Execute(ctx context.Context) (bool, error)
```

1. 生成 `Workers` 个 goroutine，从缓冲通道读取（`cap = Workers * 2`）。
2. 在调用 goroutine 上调用 `feedItems()`（生产者循环）。
3. 关闭通道，等待所有工作 goroutine 耗尽。
4. 刷新被动模块（`Flusher` 接口）和 OAST 服务。
5. 返回 `(foundResults, nil)`。

### feedItems()——生产者

对于来自 `source.Next()` 的每个项目：

1. **静态文件过滤器**：如果路径匹配静态文件扩展名（`.jpg`、`.css` 等），跳过。
2. **请求前作用域检查**：`ScopeMatcher.InScopeRequest(host, path, "", "")`——仅主机 + 路径，无 HTTP 往返。早期拒绝明显超出范围的项目。
3. **主机错误检查**：如果 `HostErrors.Check(hostID)` 返回 true（主机已被断路器断开），跳过。
4. 将项目发送到工作通道。

### worker()——消费者

每个工作 goroutine 在通道上循环：

```go
for item := range itemCh {
    e.processItem(item)
    item.Complete()
    e.statsTracker.Increment()
}
```

## 阶段 6：处理项目

`processItem()` 是每个项目的热路径。每个通过 `feedItems()` 的项目都经过以下步骤：

### 步骤 1：基线 HTTP 获取

```
if SkipBaseline && response already attached:
    使用现有响应（动态评估阶段中来自数据库的项目）
else:
    httpClient.Execute(request) → response
    在 Close() 之前从池中复制响应字节
    通过 WithResponse() 将响应附加到请求
```

响应字节从 `sync.Pool` 的回收缓冲区复制（初始 32 KiB，池返回最大 1 MiB）以减少 GC 压力。

### 步骤 2：流量回调

如果配置了，调用 `OnTraffic(method, url, statusCode, contentType)`——一个用于向 stderr 打印流量行的观察者钩子。

### 步骤 3：前置钩子

```
hooks.RunPreHooks(request)
  → error: 记录日志并跳过项目
  → nil 返回：钩子过滤掉了它，跳过项目
  → 修改后的请求：继续使用转换后的请求
```

前置钩子可以注入身份验证标头、转换请求或指示完全跳过。

### 步骤 4：正文大小强制执行

如果设置了 `ScopeMatcher`，检查请求和响应正文大小：
- `BodySizeDrop` → 完全丢弃项目。
- `BodySizeTruncate` → 将正文截断到限制，继续扫描。
- `BodySizeSkipScan` → 截断，保存到数据库，但跳过扫描。

### 步骤 5：作用域检查 + 数据库保存

```
if ScopeMatcher configured:
    检查完整作用域（主机、路径、状态、内容类型、正文字符串）
    如果超出作用域且 ScopeOnIngest：完全丢弃（不保存，不扫描）
    保存到数据库
    如果超出作用域：保存但不扫描 → 返回
else:
    保存到数据库并继续
```

`saveToDatabase()` 调用 `repo.SaveRecord()` 并将返回的 UUID 存储在 `requestUUIDs` 分片映射中（键为请求 SHA-256 哈希），用于后续的漏洞发现关联。

### 步骤 6：资格预计算

`computeEligibility()` 每个项目运行一次（而非每个模块）：
1. 请求 nil 检查
2. URL 解析检查
3. 媒体/JS URL 检查（`utils.IsMediaAndJSURL`）
4. HTTP 方法检查（跳过 `OPTIONS`、`CONNECT`、`HEAD`、`TRACE`）

缓存的 `baseEligible` 结果让执行器在基础检查会拒绝时跳过调用嵌入标准基础检查的模块的 `CanProcess()`。

### 步骤 7：模块过滤器

如果 `item.EnableModules` 非空，构建基于映射的 O(1) 过滤器。否则使用 `allModulesFilter` 哨兵。

### 步骤 8：被动模块执行（顺序）

```
runPassivePerHost(request, filter)      perHostPassive 的顺序循环
runPassivePerRequest(request, filter)   perRequestPassive 的顺序循环
```

对于每个模块：检查过滤器 → 检查 `CanProcess()` → 调用扫描方法 → 处理结果。无 goroutine——被动模块不执行网络 I/O。

### 步骤 9：主动模块执行（并行）

三个类别通过 `conc.WaitGroup` 并行运行：

```
var g conc.WaitGroup
g.Go(func() { runActivePerHost(request, filter, eligibility) })
g.Go(func() { runActivePerRequest(request, filter, eligibility) })
g.Go(func() { runActivePerInsertionPoint(request, filter, eligibility) })
g.Wait()
```

在每个类别内，符合条件的模块也并发运行（内部 `conc.WaitGroup`）。

对于插入点类别特别地，插入点**串行**迭代（一次一个），但给定点的所有符合条件的模块**并发**运行：

```
insertion points = ipCache.GetOrCompute(requestHash)
for each insertionPoint:
    for each eligible module (parallel):
        module.ScanPerInsertionPoint(request, insertionPoint, httpClient, scanCtx)
```

### 并发模型总结

```
Execute()
├── feedItems()                          [调用 goroutine，生产者]
└── Workers goroutines                   [消费者池]
    └── processItem()
        ├── 被动模块                     [工作 goroutine 上顺序]
        │   ├── runPassivePerHost
        │   └── runPassivePerRequest
        └── 主动模块                     [通过 conc.WaitGroup 3 路并行]
            ├── runActivePerHost          [内部并行：所有模块]
            ├── runActivePerRequest       [内部并行：所有模块]
            └── runActivePerInsertionPoint
                └── for each IP (serial)  [内部并行：所有模块]
```

## 阶段 7：插入点

### InsertionPoint 接口——`pkg/httpmsg/insertion_point.go`

```go
type InsertionPoint interface {
    Name() string                        // 参数名称（例如 "id"、"username"）
    BaseValue() string                   // 此位置的原始值
    Type() InsertionPointType            // INS_* 常量之一
    BuildRequest(payload []byte) []byte  // 注入载荷后的新请求字节
    PayloadOffsets(payload []byte) []int  // 构建请求中的 [startOffset, endOffset]
}
```

### InsertionPointType 常量

| 常量 | 值 | 描述 |
|---|---|---|
| `INS_PARAM_URL` | 0 | URL 查询参数值 |
| `INS_PARAM_BODY` | 1 | POST 正文参数值 |
| `INS_PARAM_COOKIE` | 2 | Cookie 值 |
| `INS_PARAM_XML` | 3 | XML 元素值 |
| `INS_PARAM_XML_ATTR` | 4 | XML 属性值 |
| `INS_PARAM_MULTIPART_ATTR` | 5 | 多部分属性值 |
| `INS_PARAM_JSON` | 6 | JSON 值 |
| `INS_HEADER` | 32 | HTTP 标头值 |
| `INS_URL_PATH_FOLDER` | 33 | REST URL 路径文件夹 |
| `INS_PARAM_NAME_URL` | 34 | URL 参数名称 |
| `INS_PARAM_NAME_BODY` | 35 | 正文参数名称 |
| `INS_ENTIRE_BODY` | 36 | 整个请求正文 |
| `INS_URL_PATH_FILENAME` | 37 | REST URL 路径文件名 |
| `INS_USER_PROVIDED` | 64 | 用户定义的位置 |
| `INS_EXTENSION_PROVIDED` | 65 | 插件提供的位置 |

### InsertionPoint 实现

| 类型 | 描述 |
|---|---|
| `ParameterInsertionPoint` | 标准参数替换。使用基于偏移的拼接，带有类型感知的载荷编码（URL 参数/正文/Cookie 使用 URL 编码，JSON 参数使用 JSON 感知编码，XML 使用原始编码）。 |
| `HeaderInsertionPoint` | 标头值替换。使用 `AddOrReplaceHeader()` 而非偏移拼接。为现有可注入标头 + 合成标头（`X-Forwarded-For`、`X-Forwarded-Host`、`Referer`、`True-Client-IP`、`X-Real-IP`）创建。 |
| `NestedInsertionPoint` | 多层编码链（例如，正文参数内 URL 编码的 JSON）。`BuildRequest()` 从内到外应用：子先构建，然后父对结果进行编码。 |
| `EncodedInsertionPoint` | 自定义编码器链。应用 `prefix + payload → encoder.Encode() → splice`。用于复杂编码场景。 |

### LRU 缓存

执行器维护一个 4096 条目的 LRU 缓存（`ipCache`），键为请求 SHA-256 哈希。`CreateAllInsertionPoints()` 对每个唯一请求调用一次，结果被所有扫描该请求的模块重用。

### 共享基础请求

`CreateAllInsertionPoints()` 创建原始字节的单个 `sharedBaseRequest` 克隆，由该调用的所有 `ParameterInsertionPoint` 实例共享。这是安全的，因为 `BuildRequest()` 从不修改共享字节——它总是分配新的结果切片。

## 阶段 8：模块分发

### 模块接口层次结构——`pkg/modules/`

```
Module（基础）
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
    └── （可选）Flusher: Flush(scanCtx)
```

### ScanScope 位掩码——`pkg/modules/modkit/types.go`

```go
const (
    ScanScopeInsertionPoint ScanScope = 1 << iota  // = 1
    ScanScopeRequest                                // = 2
    ScanScopeHost                                   // = 4
)
```

模块通过 OR 运算常量声明一个或多个作用域。执行器在启动时使用 `ScanScopes().Has(scope)` 预分组模块。

### InsertionPointTypeSet——`pkg/modules/modkit/types.go`

一个 `uint32` 位掩码，每个位对应一个 `InsertionPointType`。执行器在调用 `ScanPerInsertionPoint()` 之前检查：

```go
module.AllowedInsertionPointTypes().Contains(ip.Type())
```

预构建的预设：`URLParamTypes`、`BodyParamTypes`、`CookieTypes`、`HeaderTypes`、`AllParamTypes`。

### CanProcess 语义

**主动模块**（通过 `BaseActiveModule`）：拒绝 nil 请求、无法解析的 URL、媒体/JS URL 以及不可测试的 HTTP 方法（`OPTIONS`、`CONNECT`、`HEAD`、`TRACE`）。执行器在 `computeEligibility()` 中预计算这些检查，并在基础检查会拒绝时跳过调用 `CanProcess()`。

**被动模块**（通过 `BasePassiveModule`）：仅检查所需的 HTTP 事务部分（请求和/或响应）是否存在。它们处理所有内容类型，包括媒体——无方法过滤。

### 执行模式

```
每个项目：
  1. 被动每主机    → 顺序循环，无 goroutine
  2. 被动每请求    → 顺序循环，无 goroutine
  3. 主动每主机    → 并行：所有符合条件的模块并发
  4. 主动每请求    → 并行：所有符合条件的模块并发
  5. 主动每 IP     → 对于每个插入点（串行）：
                       所有符合条件的模块并发
```

步骤 3-5 通过 `conc.WaitGroup` 作为三个并发的 goroutine 组运行。

### ScanContext——`pkg/modules/modkit/context.go`

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

- **DedupManager**——请求级去重。
- **OASTProvider**——为盲漏洞检测生成带外回调 URL。
- **MutationGenerator**——对参数值进行分类并生成测试变异。
- **baselineCache**——缓存基线响应，用于基于差异的扫描。

### Flusher 接口

跨多个请求缓冲状态的被动模块（例如 `anomaly_ranking`）实现 `Flusher`：

```go
type Flusher interface {
    Flush(scanCtx *ScanContext)
}
```

由执行器在所有工作 goroutine 完成后调用，实现扫描结束时的聚合和最终结果输出。

### 模块开发默认值——`pkg/modules/modkit/`

模块作者嵌入 `BaseActiveModule` 或 `BasePassiveModule` 以获得所有接口方法的默认实现。模块 ID 必须是小写 kebab-case，带有前缀 `active-` 或 `passive-`（在构造时验证，违规时 panic）。`modkit` 包还提供 `NewBaseModule()`、`NewBaseActiveModule()` 和 `NewBasePassiveModule()` 构造函数。

## 阶段 9：结果输出

### ResultEvent——`pkg/output/output.go`

```go
type ResultEvent struct {
    ModuleID         string                 `json:"template-id"`
    Info             Info                   `json:"info,inline"`
    Type             string                 `json:"type"`
    Host             string                 `json:"host,omitempty"`
    URL              string                 `json:"url,omitempty"`
    Matched          string                 `json:"matched-at,omitempty"`
    ExtractedResults []string               `json:"extracted-results,omitempty"`
    Request          string                 `json:"request,omitempty"`
    Response         string                 `json:"response,omitempty"`
    Metadata         map[string]interface{} `json:"meta,omitempty"`
    Timestamp        time.Time              `json:"timestamp"`
    // ...
}
```

`ResultEvent.ID()` 计算 `ModuleID | Description | Severity | Matched` 的 SHA-1 哈希——这成为数据库中的 `finding_hash`，用于去重。

### processResults() 和 emitResult()

当模块返回结果时，执行器处理它们：

```
Module returns []*ResultEvent
  │
  ▼
processResults(results, module)
  │
  for each result:
  │
  ├── moduleFindingAllowed(module.ID())
  │     每模块漏洞发现上限检查（MaxFindingsPerModule）。
  │     当 > 0 时，达到限制后抑制结果。
  │     当模块达到上限时记录一次性警告。
  │
  ├── assignModuleInfo(result, module)
  │     设置 ModuleID、Info.Name、Description、Severity、Confidence
  │     默认 Type = "http"
  │     如果 Matched 为空则从 URL 派生
  │     如果 URL 为空则从请求字节派生
  │     从 URL 填充 Host
  │
  └── emitResult(result)
        │
        ├── 1. 后置钩子：RunPostHooks(result)
        │      nil 返回 → 丢弃结果（钩子过滤掉了它）
        │
        ├── 2. 设置结果标志：e.results.Store(true)
        │
        ├── 3. 数据库保存：
        │      从 result.Request 构建临时 HttpRequest
        │      查找 requestUUIDs[requestHash] → recordUUID
        │      repo.SaveFinding(result, [recordUUID], scanUUID)
        │      使用 INSERT ON CONFLICT (finding_hash) DO NOTHING
        │
        ├── 4. OnResult 回调 → 输出写入器
        │
        └── 5. Notifier.Send(result) → Telegram/Discord
```

## 阶段 10：输出

### Writer 接口——`pkg/output/output.go`

```go
type Writer interface {
    Close()
    Write(*ResultEvent) error
    WriteFileOnly(*ResultEvent) error
}
```

### StandardWriter

默认的 `Writer` 实现：

1. 设置 `Timestamp = time.Now()`，默认 `Type = "http"`，强制 `MatcherStatus = true`。
2. 通过 `jsoniter.Marshal()` 序列化为 JSON。
3. 在互斥锁下：
   - **Stdout**：写入 JSON（如果 `--json`）或格式化的控制台输出（如果不是 `--silent`）。
   - **文件**：将 JSON 行追加到输出文件（JSONL 格式）。

### 控制台格式——`pkg/output/format_screen.go`

```
[› phase │] [moduleType] [moduleName] [severity] matched-at [extracted-results] [fuzz-param]
```

- 模块 ID 拆分为类型（`active`/`passive`）和名称，相应着色。
- 严重性以符号和 ANSI 颜色显示（Critical=品红、High=红色、Medium=黄色、Low=绿色）。
- 输出截断到终端宽度。

### JSON 格式——`pkg/output/format_json.go`

通过 `jsoniter.Marshal()` 序列化 `ResultEvent`。除非设置了 `--include-response`，否则响应体会被剥离。

### HTML 格式——`pkg/output/format_html.go`

使用流式方法：在 `{{.ResultsJSON}}` 处分割嵌入式 HTML 模板，使用简单的字符串替换写入前部分（避免 `text/template`，因为捆绑的 JS 包含 `{{` 序列），然后一次一个地流式传输 JSON 数组项，然后写入后部分。

### 文件输出写入器——`pkg/output/file_output_writer.go`

```go
type fileWriter struct {
    file *os.File
    mu   sync.Mutex
}
```

互斥锁保护，追加 JSON + 换行（JSONL 格式）。使用 `O_APPEND|O_CREATE|O_WRONLY` 打开，以便跨调用安全恢复。

## 阶段 11：数据库持久化

### 数据模型——`pkg/database/models.go`

#### HTTPRecord（表：`http_records`）

完全反规范化——没有单独的主机或参数表。关键字段：

- **标识**：`UUID`（主键）、`RequestHash`（原始请求的 SHA-256）
- **主机信息**：`Scheme`、`Hostname`、`Port`、`IP`
- **请求**：`Method`、`Path`、`URL`、`RequestHeaders`（JSONB）、`RawRequest`（bytea）、`RequestBody`（bytea）
- **响应**：`StatusCode`、`ResponseHeaders`（JSONB）、`RawResponse`（bytea）、`ResponseBody`（bytea）、`ResponseTitle`、`ResponseWords`
- **参数**：`Parameters`（`EmbeddedParam` 的 JSONB 数组）
- **风险**：`RiskScore`、`Remarks`（JSONB 数组）
- **元数据**：`Source`、`SentAt`、`ReceivedAt`、`CreatedAt`

#### Finding（表：`findings`）

- **标识**：`ID`（自增）、`FindingHash`（用于去重的唯一约束）
- **模块信息**：`ModuleID`、`ModuleName`、`Description`、`Severity`、`Confidence`
- **匹配数据**：`MatchedAt`（JSONB 数组）、`ExtractedResults`、`Request`、`Response`
- **关系**：`HTTPRecordUUIDs`（JSONB 数组）、`ScanUUID`
- **分组证据**：`AdditionalEvidence`（字符串的 JSONB 数组）——来自合并到此幸存者中的重复漏洞发现的请求/响应对（上限 10 条）

`finding_records` 关联表将漏洞发现链接到 HTTP 记录（多对多）。

### 转换器——`pkg/database/converters.go`

- `HTTPRecord.FromHttpRequestResponse()`——将内存中的类型转换为数据库模型。生成 UUID、解析 URL、复制标头/正文、计算哈希、提取 HTML 标题、统计响应字数。
- `Finding.FromResultEvent()`——将 `ResultEvent` 字段映射到 `Finding`。设置 `FindingHash = event.ID()`（SHA-1 去重哈希）。

### Repository——`pkg/database/repository.go`

关键方法：

| 方法 | 描述 |
|---|---|
| `SaveRecord()` | 单条 INSERT，返回 UUID |
| `SaveRecordsBatch()` | 在单个事务中批量 INSERT |
| `SaveFinding()` | INSERT ON CONFLICT (finding_hash) DO NOTHING + 证据追加 + 关联表 |
| `DeduplicateFindings()` | 阶段后分组：合并共享 (module_id, severity, matched_at URL) 的漏洞发现 |
| `CreateScanWithCursor()` | 创建扫描记录，从上次完成的扫描复制游标 |
| `CountRecordsAfterCursor()` | 统计游标后的新记录数（用于反馈循环） |
| `GetRecordsWithResponseBody()` | UUID 游标分页，用于批量扫描（Kingfisher） |
| `UpdateRiskScores()` | 批量 CASE/WHEN UPDATE，每条语句 500 个 UUID |

### RecordWriter——`pkg/database/record_writer.go`

用于高吞吐量导入的批量异步持久化：

```go
type RecordWriter struct {
    repo    *Repository
    cfg     RecordWriterConfig   // BufferSize=4096, BatchSize=128, FlushInterval=50ms
    ch      chan writeRequest     // 通过通道容量实现背压
}
```

- `Write()` 转换为 `HTTPRecord`，发送到缓冲通道，阻塞直到刷新。
- `flushLoop()` 作为单个后台 goroutine 运行：累积批次，在批次满或计时器触发时通过 `repo.SaveRecordsBatch()` 刷新。
- 每个调用者通过每个请求的结果通道获得 `WriteResult{UUID, Err}`。

## 阶段 12：支持系统

### 作用域匹配——`internal/config/scope_matcher.go`

`ScopeMatcher` 根据可配置规则跨多个维度评估项目（全部 AND 运算）：

1. **主机**：glob 匹配 + 来源模式过滤（按主机缓存）
2. **路径**：`filepath.Match` glob 模式
3. **静态文件扩展名**：可配置的扩展名集合
4. **状态码**：精确、通配符（`2xx`）或范围（`400-499`）
5. **内容类型**：请求和响应的 glob 模式
6. **正文字符串**：请求/响应正文上的不区分大小写的子串匹配

**来源模式**控制 CLI 目标如何约束主机作用域：

| 模式 | 匹配规则 |
|---|---|
| `all` | 无限制 |
| `strict` | 精确主机名匹配 |
| `balanced` | eTLD+1 必须匹配（例如 `*.example.com`） |
| `relaxed`（默认） | 主机包含目标关键词 |

### 速率限制——`pkg/core/ratelimit/host_limiter.go`

`HostRateLimiter` 提供每主机并发控制：

- **32 个固定分片**，使用内联 FNV-1a 哈希进行分片选择。
- 每个主机获得一个**缓冲通道信号量**（容量 = `MaxPerHost`，默认 2）。
- `Acquire(ctx, host)` 阻塞直到有可用槽位；`Release(host)` 释放槽位。
- 后台驱逐 goroutine 移除空闲条目（默认：30 秒空闲，每 10 秒检查一次）。
- 每分片容量上限，超出时驱逐最旧条目。

### 主机错误断路器——`pkg/core/hosterrors/`

`hosterrors.Cache` 跟踪每个主机的连续错误：

- `MarkFailed()` 增加错误计数器（带基于正则的错误匹配）。
- `Check()` 在计数器达到 `MaxHostError`（默认 30）时返回 true。
- `MarkSuccess()` 重置计数器（但如果已达到阈值则不重置）。
- 执行器的 `feedItems()` 预检查此项，以跳过隔离主机的项目。

### JS 插件钩子——`pkg/jsext/hooks.go`

**前置钩子**（`PreHookExecutor`）：在模块分发之前转换或过滤请求。返回 `nil` 以跳过项目。

**后置钩子**（`PostHookExecutor`）：在输出之前转换或过滤结果。返回 `nil` 以丢弃结果。

`HookChain` 顺序执行钩子，将每个钩子的输出传递给下一个。出错时，跳过该钩子（非致命）。返回 `nil` 时，链立即中止。

每个钩子使用一个 `VMPool`（Sobek VM 的 `sync.Pool`）——VM 在并发调用之间重用，没有共享的可变状态。

### OAST（带外）

用于盲漏洞（SSRF、XXE 等）的带外回调检测。OAST 服务为每个模块/参数/请求生成唯一的回调 URL，并在扫描结束时刷新，带有宽限期以捕获延迟的回调。

### 去重和漏洞发现分组

三个级别的去重防止噪音和冗余：

1. **请求级**：`DedupManager` 防止扫描重复请求（在模块分发前检查）。

2. **漏洞发现级（内联）**：数据库中的 `finding_hash` 唯一约束使用 `INSERT ON CONFLICT DO NOTHING`。在插入时检测到重复哈希时，`appendRecordsToFinding()` 将新的 HTTP 记录 UUID 和请求/响应对（作为 `AdditionalEvidence`）追加到现有漏洞发现，而不是创建新行。

3. **漏洞发现级（阶段后分组）**：`DeduplicateFindings()` 在 KnownIssueScan 和动态评估阶段之后运行。它对项目内共享相同 `(module_id, severity, matched_at[0] URL)` 的漏洞发现进行分组——这捕获了同一模块使用不同载荷在同一 URL 上多次触发的情况（例如，注入探测在每个端点上产生数十个结果）。

   分组过程：
   - 按 `module_id || severity || matched_at[0]` 分区漏洞发现，并按 `created_at ASC` 排序
   - 保留每组中最早的漏洞发现作为**幸存者**
   - 将重复项的请求/响应对收集到幸存者的 `AdditionalEvidence` 字段中（上限 10 条以控制存储）
   - 删除所有重复的漏洞发现及其 `finding_records` 关联行
   - 返回已删除漏洞发现和已合并组的计数，用于用户反馈

4. **漏洞发现级（值分组）**：`GroupFindingsByValue()` 作为第二个阶段后遍历运行（由 `known_issue_scan.group_by_value` 控制，默认开启）。它折叠跨许多 URL 重复**相同提取值**的漏洞发现——键为 `(module_id, severity[, hostname], normalized extracted_results)`——因此一个在数十个页面上出现的泄露密钥成为单个漏洞发现。`by_module` 中列出的模块是更强的情况：它们按 `(module_id, severity[, hostname])` 折叠，**不考虑**每个 URL 的值，适用于每个资产触发一次的模块，其中不同的值是噪音而非信号（例如 `sourcemap-detect`，每个 bundle 一个 `.map` 文件名；`unsafe-html-sink` 和源分析引导系列，每个 JS 文件一个片段上下文；`cookie-security-detect`，每个 Set-Cookie 响应一个漏洞发现）。分组默认按主机进行（`per_host`），因此两个主机名上的相同值保持为两个漏洞发现，幸存者的 `matched_at` 保留每个受影响的 URL，最多 `max_urls`。携带密钥的模块（`env-secret-exposure`）被排除在 `by_module` 之外，以便不同的泄露值保持为不同的漏洞发现。

```
阶段完成（KnownIssueScan 或动态评估）
  │
  ▼
DeduplicateFindings(projectUUID)
  │
  ├── GROUP BY (module_id, severity, matched_at[0])
  │     ORDER BY created_at ASC → survivor = row_number 1
  │
  ├── 对于每个有重复的组：
  │     合并重复的请求/响应 → survivor.AdditionalEvidence
  │     上限 10 条证据
  │
  ├── DELETE 重复的漏洞发现 + 关联行
  │
  └── 打印反馈："grouped N findings into M"
```

## 综合

### 端到端流程

```
vigolium scan -t https://example.com
         │
         ▼
    ┌────────────┐
    │  CLI 解析   │  pkg/cli/scan.go: runScanCmd()
    │  + 配置     │  加载设置，解析策略/配置文件
    │  + 数据库初始化│  database.NewDB() → CreateSchema()
    └─────┬──────┘
          │
          ▼
    ┌────────────┐
    │   运行器    │  internal/runner/runner.go
    │  构建基础设施│  HTTP 客户端、作用域匹配器、速率限制器、钩子
    └─────┬──────┘
          │
          ▼
    ┌────────────────────────────────────────────────────────┐
    │              RunNativeScan() — 6 阶段                   │
    │                                                        │
    │  [启发式] → [收集] → [爬取]                            │
    │       → [发现/导入] → [KnownIssueScan] → [动态评估]   │
    │                                                        │
    │  阶段 0-4：用 HTTP 记录填充数据库                       │
    │  阶段 4-5：每个阶段后执行 DeduplicateFindings()        │
    │  阶段 5：使用模块扫描记录                               │
    └───────────────────────┬────────────────────────────────┘
                            │
                            ▼  （阶段 6 详情）
    ┌───────────────────────────────────────────────────┐
    │                    执行器                           │
    │                                                   │
    │  feedItems():                                     │
    │    source.Next() → 静态过滤器 → 作用域检查        │
    │    → 主机错误检查 → 发送到工作通道                │
    │                                                   │
    │  worker() → processItem():                        │
    │    1. 基线 HTTP 获取（或使用数据库响应）          │
    │    2. 流量回调                                    │
    │    3. 前置钩子（JS 转换/过滤）                    │
    │    4. 正文大小强制执行                            │
    │    5. 作用域检查 + 数据库保存                     │
    │    6. 资格预计算                                  │
    │    7. 被动模块（顺序）                            │
    │    8. 主动模块（并行，3 路）                      │
    │       └── 每个插入点：所有模块                    │
    │                                                   │
    │  后处理：                                         │
    │    刷新被动模块（Flusher 接口）                   │
    │    刷新 OAST 服务（宽限期）                       │
    └───────────────────────┬───────────────────────────┘
                            │
                            ▼
    ┌───────────────────────────────────────────────────┐
    │              结果输出                              │
    │                                                   │
    │  每模块漏洞发现上限（达到限制后抑制）               │
    │  assignModuleInfo() → emitResult():               │
    │    1. 后置钩子（JS 转换/过滤）                    │
    │    2. SaveFinding() 到数据库（通过 finding_hash   │
    │       去重 + 冲突时证据追加）                     │
    │    3. OnResult → StandardWriter.Write()           │
    │    4. Notifier.Send() → Telegram/Discord          │
    └───────────────────────┬───────────────────────────┘
                            │
                            ▼
    ┌───────────────────────────────────────────────────┐
    │                   输出                              │
    │                                                   │
    │  控制台：彩色严重性 + 模块 + 匹配 URL             │
    │  JSON：   通过 jsoniter 的 JSONL                   │
    │  HTML：   嵌入式 ag-grid 模板                     │
    │  文件：   带互斥锁的追加型 JSONL                   │
    └───────────────────────────────────────────────────┘
```

### 汇总表

| 阶段 | 关键文件 | 关键函数 | 输入数据 | 输出数据 |
|---|---|---|---|---|
| CLI 入口 | `cmd/vigolium/main.go` | `main()` → `cli.Execute()` | CLI 参数 | — |
| 配置 | `pkg/cli/scan.go` | `runScanCmd()` | 标志 + YAML | `*types.Options`、`*config.Settings` |
| 输入 | `pkg/input/source/` | `InputSource.Next()` | URL/文件/stdin | `*work.WorkItem` |
| HTTP 类型 | `pkg/httpmsg/` | `GetRawRequestFromURL()` | URL 字符串 | `*HttpRequestResponse` |
| 运行器 | `internal/runner/runner.go` | `RunNativeScan()` | Options + Settings | 阶段结果 |
| 执行器 | `pkg/core/executor.go` | `Execute()` → `processItem()` | `InputSource` + 模块 | `bool`（是否发现结果） |
| 插入点 | `pkg/httpmsg/insertion_point.go` | `CreateAllInsertionPoints()` | 原始请求字节 | `[]InsertionPoint` |
| 模块分发 | `pkg/modules/` | `ScanPer{Host,Request,InsertionPoint}()` | `*HttpRequestResponse` | `[]*ResultEvent` |
| 结果输出 | `pkg/core/executor.go` | `emitResult()` | `*ResultEvent` | 数据库写入 + 输出 |
| 输出 | `pkg/output/output.go` | `StandardWriter.Write()` | `*ResultEvent` | 控制台/JSON/HTML/文件 |
| 数据库持久化 | `pkg/database/` | `SaveRecord()`、`SaveFinding()` | HTTP 类型 / ResultEvent | `HTTPRecord`、`Finding` |

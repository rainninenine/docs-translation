# Native Scan Phases

本地扫描（Native Scan）通过多个阶段处理目标，每个阶段负责扫描过程的一个特定部分。理解这些阶段有助于您优化扫描、排除问题，并根据需要运行特定阶段。

## 阶段概览

```
Phase 0: Heuristics    → 分析目标，选择策略
Phase 1: Harvest       → 从公开来源收集信息
Phase 2: Spidering     → 浏览器爬取（可选）
Phase 3: Discovery     → 内容发现（目录枚举）
Phase 4: Ingestion     → 导入外部流量
Phase 5: Dynamic Assessment → 主动漏洞扫描
Phase 6: Known Issue Scan  → CVE 和密钥检测
```

## 阶段详情

### Phase 0: Heuristics（启发式分析）

分析目标并选择适当的扫描策略。确定目标使用的技术栈（例如 React、Angular、PHP、Node.js）。

### Phase 1: Harvest（信息收集）

从公开来源收集关于目标的信息。这包括检查 `robots.txt`、`sitemap.xml`、`security.txt` 以及常见的元数据文件。

### Phase 2: Spidering（浏览器爬取）

使用 Spitolas（基于 Chromium 的爬虫）通过浏览器交互发现 SPA 路由和动态内容。此阶段可选，由 `--spider` 标志控制。

详见 [Spidering](../native-scan/phases/spidering.md)。

### Phase 3: Discovery（内容发现）

使用 Deparos 引擎进行智能内容发现——目录枚举、端点发现和文件模糊测试。它从每个响应中学习，自适应调整策略，并通过指纹识别过滤误报。

详见 [Discovery](../native-scan/phases/discovery.md)。

### Phase 4: Ingestion（数据导入）

导入外部流量来源——OpenAPI 规范、Postman 集合、Burp 导出文件、原始 HTTP 请求和 curl 命令。

详见 [Ingestion](../server-mode/ingestion.md)。

### Phase 5: Dynamic Assessment（动态评估）

核心扫描阶段。主动模块向每个插入点注入载荷，被动模块分析现有流量。这是发现漏洞的主要阶段。

详见 [Dynamic Assessment](../native-scan/phases/dynamic-assessment.md)。

### Phase 6: Known Issue Scan（已知问题扫描）

使用 Nuclei 模板检测已知 CVE 和常见配置错误，并使用 Kingfisher 引擎扫描暴露的密钥。

详见 [Known Issue Scan](../native-scan/phases/known-issue-scan.md)。

## 运行特定阶段

```bash
# 仅运行发现阶段
vigolium run discovery -t https://example.com

# 仅运行动态评估
vigolium run dynamic-assessment -t https://example.com

# 仅运行扩展
vigolium run extension -t https://example.com --ext my-scanner.js

# 跳过特定阶段
vigolium scan -t https://example.com --skip spidering

# 仅运行特定阶段
vigolium scan -t https://example.com --only dynamic-assessment
```

## 阶段依赖关系

某些阶段依赖于先前阶段的数据：

| 阶段 | 依赖 | 原因 |
|------|------|------|
| Spidering | Heuristics | 需要知道目标类型 |
| Discovery | Spidering（可选） | 从爬虫发现的路径开始 |
| Dynamic Assessment | 所有先前阶段 | 需要要扫描的 HTTP 记录 |
| Known Issue Scan | Dynamic Assessment | 使用所有发现的路径 |

## 延伸阅读

- [Deparos — Content Discovery](../native-scan/phases/discovery.md) — 发现阶段详情
- [Spitolas — Browser Crawler](../native-scan/phases/spidering.md) — 爬虫详情
- [Dynamic Assessment](../native-scan/phases/dynamic-assessment.md) — 核心扫描阶段
- [Known Issue Scan](../native-scan/phases/known-issue-scan.md) — CVE 和密钥检测
- [Extension Scanning](../native-scan/phases/extension.md) — 自定义扩展

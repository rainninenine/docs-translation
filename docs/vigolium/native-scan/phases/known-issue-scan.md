# KnownIssueScan — Known Vulnerability and Secret Detection

KnownIssueScan 使用 Nuclei 模板和 Kingfisher 密钥检测引擎检查目标是否存在已知 CVE、常见配置错误以及暴露的密钥。它在内容发现阶段之后运行，利用之前阶段发现的所有路径和端点来最大化覆盖范围。

## Why KnownIssueScan Matters

许多真实世界的入侵事件利用了未修补的公开披露漏洞（CVE），或意外提交到响应体中的密钥。KnownIssueScan 在整个已发现的攻击面上系统性地测试这些已知问题——捕获那些基于自定义模糊测试的模块无法检测到的低垂果实。

## How It Works

```
Stored HTTP Records (from phases 0-4)
  │
  ▼
┌─────────────────────────────────────────────────┐
│  Target Enrichment                               │
│  • GetDistinctPaths() from database              │
│  • Enrich targets with discovered paths          │
│    (enrich_targets: true by default)             │
│  • Or host-level only when disabled              │
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│  Nuclei Template Engine                          │
│  • CVE detection (known vulnerability checks)    │
│  • Misconfiguration detection                    │
│  • Technology fingerprinting                     │
│  • Custom template support (templates_dir)       │
│  • Tag-based filtering (include/exclude)         │
│  • Severity filtering (critical → info)          │
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│  Kingfisher Secret Detection                     │
│  • Scans stored response bodies                  │
│  • Detects API keys, tokens, credentials         │
│  • Filters out secret_detect passive module      │
│    to avoid duplicates in dynamic-assessment phase│
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│  Post-Phase Processing                           │
│  DeduplicateFindings() groups findings with      │
│  identical (module_id, severity, matched_at URL) │
└─────────────────────────────────────────────────┘
```

## Configuration

KnownIssueScan 在 `vigolium-configs.yaml` 中的 `known_issue_scan` 键下配置：

```yaml
known_issue_scan:
  tags: []              # nuclei template tags to include (empty = all)
  exclude_tags: [dos]   # tags to exclude (default: dos)
  severities: []        # filter by severity: critical, high, medium, low, info (empty = all)
  templates_dir: ""     # custom templates directory (empty = built-in)
  enrich_targets: true  # enrich targets with paths from previous phases
  severity_overrides:   # remap a finding's severity by template ID (case-insensitive)
    config-json-exposure-fuzz: medium
```

### Key Options

| 选项 | 默认值 | 描述 |
|------|--------|------|
| `tags` | `[]` (全部) | 仅包含匹配这些标签的模板 |
| `exclude_tags` | `[dos]` | 排除匹配这些标签的模板 |
| `severities` | `[]` (全部) | 按严重级别过滤结果 |
| `templates_dir` | 内置 | 自定义 Nuclei 模板的路径 |
| `enrich_targets` | `true` | 将发现的路径附加到目标 URL 以获得更广的覆盖范围 |
| `severity_overrides` | `{config-json-exposure-fuzz: medium}` | 按 nuclei 模板 ID 重新映射漏洞发现的记录严重级别。在输出/持久化之前应用（输出、计数和存储的漏洞发现均一致）。允许您调整噪声较大或上下文相关的模板的严重级别，而无需分叉上游模板（上游模板会在 `nuclei -update-templates` 时恢复）。将条目设置回模板的原始严重级别可撤销默认的重新映射。 |

## Runtime Defaults

| 参数 | 默认值 |
|-------|---------|
| 并发数 | 50 |
| 速率限制 | 100 req/s |
| 超时时间 | 30 分钟 |

## Phase Execution Detail

1. 通过 `GetDistinctPaths()` 从数据库查询不同的路径。
2. 构建目标 URL——可以是路径增强的（默认，`enrich_targets: true`）或仅主机级别。
3. 使用配置的并发数和速率限制对增强后的目标运行 Nuclei 模板。
4. 对存储的响应体运行 Kingfisher 密钥扫描。
5. 每个漏洞发现以 `ModuleType: "known-issue-scan"` 和 `FindingSource: "known-issue-scan"` 保存到数据库。
6. **阶段后去重**：调用 `DeduplicateFindings()` 对具有相同 `(module_id, severity, matched_at URL)` 的漏洞发现进行分组。

## CLI Usage

仅运行 KnownIssueScan 阶段：

```bash
vigolium scan --url https://example.com --only known-issue-scan
```

跳过 KnownIssueScan 阶段：

```bash
vigolium scan --url https://example.com --skip known-issue-scan
```

## Integration

KnownIssueScan 作为本地扫描流水线中的第 6 阶段运行，位于动态评估之后。它使用之前阶段存储的 HTTP 记录和发现的路径。将 known-issue-scan 放在最后运行，可以避免其高流量的 Nuclei/Kingfisher 流量在主动/被动模块扫描完成之前触发主机速率限制（429）。

```
Discovery (Phase 4)
  → Paths and records stored in DB
  → DynamicAssessment (Phase 5)
    → Active + passive scanner modules
  → KnownIssueScan (Phase 6)
    → Nuclei templates + Kingfisher secrets
    → DeduplicateFindings()
```
# Stateless Scan

无状态扫描（Stateless Scan）是一种快速、一次性的扫描模式，不持久化任何数据。它非常适合 CI/CD 流水线、快速安全检查以及任何不需要长期存储结果的场景。

## 何时使用

| 场景 | 推荐方式 |
|------|----------|
| CI/CD 流水线 | 无状态扫描 |
| 快速安全检查 | 无状态扫描 |
| 探索性测试 | 无状态扫描 |
| 深度审计 | 标准（有状态）扫描 |
| 需要历史数据 | 标准（有状态）扫描 |

## 使用方法

```bash
# 基本无状态扫描
vigolium scan -t https://example.com --stateless

# 带 JSON 输出的无状态扫描
vigolium scan -t https://example.com --stateless --json

# 输出到文件
vigolium scan -t https://example.com --stateless --json -o results.json

# 静默模式（仅输出结果）
vigolium scan -t https://example.com --stateless --silent --json -o results.json
```

## 工作原理

当使用 `--stateless` 时，Vigolium：

1. **创建临时数据库** — 在内存中创建 SQLite 数据库
2. **运行扫描** — 像往常一样执行所有扫描阶段
3. **输出结果** — 将结果写入 stdout 或文件
4. **清理** — 丢弃临时数据库

这意味着每次无状态扫描都是完全独立的——没有数据在运行之间持久化。

## 与有状态扫描对比

| 方面 | 无状态 | 有状态 |
|------|--------|--------|
| 数据库 | 内存中（临时） | 磁盘上（持久化） |
| 运行间持久化 | 否 | 是 |
| 速度 | 更快（无 I/O） | 较慢（有 I/O） |
| 结果查询 | 仅输出文件 | 数据库查询 |
| 增量扫描 | 否 | 是 |
| 项目隔离 | 否 | 是 |

## 输出格式

### JSON 输出

```bash
vigolium scan -t https://example.com --stateless --json
```

输出示例：
```json
{"template-id":"xss-reflected","type":"http","host":"example.com","url":"https://example.com/search?q=test","matched-at":"<script>alert(1)</script>","severity":"high","timestamp":"2024-01-01T00:00:00Z"}
{"template-id":"sqli-error","type":"http","host":"example.com","url":"https://example.com/users?id=1","matched-at":"SQL syntax error","severity":"critical","timestamp":"2024-01-01T00:00:01Z"}
```

### HTML 输出

```bash
vigolium scan -t https://example.com --stateless --html -o report.html
```

生成一个包含所有发现的交互式 HTML 报告。

## 在 CI/CD 中使用

无状态扫描是 CI/CD 集成的推荐模式：

```yaml
# GitHub Actions 示例
- name: Security Scan
  run: |
    vigolium scan -t $URL --stateless --json -o results.json
    if jq -e '.[] | select(.severity == "critical")' results.json > /dev/null; then
      exit 1
    fi
```

详见 [CI/CD Integration](ci-cd-integration.md)。

## 限制

- **无历史数据** — 您无法查询过去的扫描结果
- **无增量扫描** — 每次扫描从头开始
- **无项目隔离** — 所有结果在单次运行中
- **内存使用** — 大型扫描可能消耗大量内存

## 延伸阅读

- [CI/CD Integration](ci-cd-integration.md) — 流水线集成
- [Scanning an API](scanning-an-api.md) — API 扫描指南
- [Native Scan Phases](native-scan-phases.md) — 扫描阶段详解

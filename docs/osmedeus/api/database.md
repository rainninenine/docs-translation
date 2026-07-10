---
title: "数据库管理"
description: "用于数据库管理的管理端点"
---

# 数据库管理

用于管理数据库的管理端点。

## 列出数据库表

获取所有数据库表及其行数的列表。

```bash
curl http://localhost:8002/osm/api/database/tables \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    {"name": "runs", "rows": 150},
    {"name": "step_results", "rows": 2500},
    {"name": "assets", "rows": 5000},
    {"name": "vulnerabilities", "rows": 120},
    {"name": "workspaces", "rows": 50},
    {"name": "artifacts", "rows": 800},
    {"name": "event_logs", "rows": 1200},
    {"name": "schedules", "rows": 8}
  ]
}
```

---

## 清空数据库表

清空指定数据库表中的所有记录。请谨慎使用。

```bash
curl -X POST http://localhost:8002/osm/api/database/tables/runs/clear \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Table cleared successfully",
  "table": "runs",
  "rows_deleted": 150
}
```

**错误响应（无效表名）：**
```json
{
  "error": true,
  "message": "Invalid table name: unknown_table"
}
```

**有效表名：**
- `runs` - 工作流运行记录
- `step_results` - 步骤执行结果
- `assets` - 已发现的资产
- `vulnerabilities` - 漏洞记录
- `workspaces` - 项目空间元数据
- `artifacts` - 输出产物
- `event_logs` - 事件日志记录
- `schedules` - 定时工作流
- `asset_diff_snapshots` - 资产差异快照
- `vuln_diff_snapshots` - 漏洞差异快照

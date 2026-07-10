---
title: "系统统计"
description: "聚合系统统计信息"
---

# 系统统计

## 获取系统统计

获取聚合系统统计信息，包括工作流、运行实例、项目空间、资产、漏洞和调度。

```bash
curl http://localhost:8002/osm/api/stats \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "workflows": {
    "total": 25,
    "flows": 10,
    "modules": 15
  },
  "runs": {
    "total": 150,
    "completed": 120,
    "running": 5,
    "failed": 10,
    "pending": 15
  },
  "workspaces": {
    "total": 50
  },
  "assets": {
    "total": 5000
  },
  "vulnerabilities": {
    "total": 150,
    "critical": 10,
    "high": 25,
    "medium": 50,
    "low": 65
  },
  "schedules": {
    "total": 8,
    "enabled": 5
  }
}
```

**统计字段：**

| 分类 | 字段 | 描述 |
|----------|-------|-------------|
| workflows.total | int | 工作流总数（flows + modules） |
| workflows.flows | int | flow 类型工作流数量 |
| workflows.modules | int | module 类型工作流数量 |
| runs.total | int | 运行总数 |
| runs.completed | int | 成功完成的运行 |
| runs.running | int | 当前正在运行的工作流 |
| runs.failed | int | 失败的运行 |
| runs.pending | int | 等待开始的待处理运行 |
| workspaces.total | int | 扫描项目空间总数 |
| assets.total | int | 所有项目空间中发现的资产总数 |
| vulnerabilities.total | int | 漏洞总数（所有严重级别之和） |
| vulnerabilities.critical | int | 严重级别漏洞 |
| vulnerabilities.high | int | 高级别漏洞 |
| vulnerabilities.medium | int | 中级别漏洞 |
| vulnerabilities.low | int | 低级别漏洞 |
| schedules.total | int | 已配置的调度总数 |
| schedules.enabled | int | 当前启用的调度数 |

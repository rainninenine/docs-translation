---
title: "项目空间"
description: "列出和管理扫描项目空间"
---

# 项目空间

## 列出项目空间

获取所有运行项目空间的列表及完整元数据。

**从数据库列出项目空间（默认）：**
```bash
curl http://localhost:8002/osm/api/workspaces \
  -H "Authorization: Bearer ***"
```

**带分页列出项目空间：**
```bash
curl "http://localhost:8002/osm/api/workspaces?offset=0&limit=50" \
  -H "Authorization: Bearer ***"
```

**从文件系统/资产列出项目空间：**
```bash
curl "http://localhost:8002/osm/api/workspaces?filesystem=true" \
  -H "Authorization: Bearer ***"
```

**组合分页与文件系统模式：**
```bash
curl "http://localhost:8002/osm/api/workspaces?filesystem=true&offset=20&limit=10" \
  -H "Authorization: Bearer ***"
```

**响应（数据库模式）：**
```json
{
  "data": [
    {
      "id": 1,
      "name": "example.com",
      "local_path": "/home/user/osmedeus-base/workspaces/example.com",
      "total_assets": 150,
      "total_subdomains": 120,
      "total_urls": 500,
      "total_vulns": 12,
      "vuln_critical": 2,
      "vuln_high": 3,
      "vuln_medium": 4,
      "vuln_low": 3,
      "vuln_potential": 0,
      "risk_score": 7.5,
      "tags": ["production", "priority"],
      "last_run": "2025-01-15T10:30:00Z",
      "run_workflow": "subdomain-enum",
      "created_at": "2025-01-10T08:00:00Z",
      "updated_at": "2025-01-15T10:30:00Z"
    },
    {
      "id": 2,
      "name": "test.com",
      "local_path": "/home/user/osmedeus-base/workspaces/test.com",
      "total_assets": 50,
      "total_subdomains": 35,
      "total_urls": 120,
      "total_vulns": 3,
      "vuln_critical": 0,
      "vuln_high": 1,
      "vuln_medium": 2,
      "vuln_low": 0,
      "vuln_potential": 5,
      "risk_score": 4.2,
      "tags": ["staging"],
      "last_run": "2025-01-14T15:00:00Z",
      "run_workflow": "port-scan",
      "created_at": "2025-01-12T12:00:00Z",
      "updated_at": "2025-01-14T15:00:00Z"
    }
  ],
  "pagination": {
    "total": 100,
    "offset": 0,
    "limit": 20
  }
}
```

**项目空间字段参考：**

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `id` | int | 唯一项目空间标识符 |
| `name` | string | 项目空间名称（通常是目标域名） |
| `local_path` | string | 项目空间目录的完整路径 |
| `total_assets` | int | 发现的资产总数 |
| `total_subdomains` | int | 发现的子域名总数 |
| `total_urls` | int | 发现的 URL 总数 |
| `total_vulns` | int | 发现的漏洞总数 |
| `vuln_critical` | int | 严重级别漏洞 |
| `vuln_high` | int | 高级别漏洞 |
| `vuln_medium` | int | 中级别漏洞 |
| `vuln_low` | int | 低级别漏洞 |
| `vuln_potential` | int | 潜在/信息性发现 |
| `risk_score` | float | 计算的风险评分（0-10） |
| `tags` | array | 用于组织的自定义标签 |
| `last_run` | timestamp | 上次工作流运行时间戳 |
| `run_workflow` | string | 最后执行的工作流名称 |
| `state_execution_log` | string | 执行日志文件路径 |
| `state_completed_file` | string | 完成标记文件路径 |
| `state_workflow_file` | string | 工作流 YAML 文件路径 |
| `state_workflow_folder` | string | 工作流文件夹路径 |
| `created_at` | timestamp | 项目空间创建时间戳 |
| `updated_at` | timestamp | 最后更新时间戳 |

---

## 列出项目空间名称

获取仅包含项目空间名称的轻量级列表（不含完整元数据）。

```bash
curl http://localhost:8002/osm/api/workspace-names \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    "example.com",
    "test.com",
    "demo.org"
  ],
  "count": 3
}
```

---

## 获取项目空间状态文件

检索项目空间状态文件的内容。此端点提供对与项目空间关联的执行日志、完成标记和工作流文件的访问。

**获取执行日志：**
```bash
curl "http://localhost:8002/osm/api/workspaces/example.com/state-file?state_file=execution_log" \
  -H "Authorization: Bearer ***"
```

**获取完成文件：**
```bash
curl "http://localhost:8002/osm/api/workspaces/example.com/state-file?state_file=completed_file" \
  -H "Authorization: Bearer ***"
```

**获取工作流文件：**
```bash
curl "http://localhost:8002/osm/api/workspaces/example.com/state-file?state_file=workflow_file" \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "workspace": "example.com",
  "state_file": "execution_log",
  "file_path": "/home/user/osmedeus-base/workspaces/example.com/log/execution.log",
  "content": "2025-01-15 10:30:00 [INFO] Starting workflow...\n2025-01-15 10:30:05 [INFO] Step 1 completed...\n..."
}
```

**状态文件类型：**

| 类型 | 描述 |
|------|-------------|
| `execution_log` | 包含时间戳和步骤输出的详细执行日志 |
| `completed_file` | 指示工作流完成状态的标记文件 |
| `workflow_file` | 已执行的 YAML 工作流定义 |

**错误响应：**

| 状态码 | 描述 |
|--------|-------------|
| 400 | 无效的项目空间名称或缺少 state_file 参数 |
| 403 | 检测到路径遍历尝试 |
| 404 | 未找到项目空间或状态文件 |

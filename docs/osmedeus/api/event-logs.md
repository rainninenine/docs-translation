---
title: "事件日志"
description: "带过滤功能的执行事件日志查询"
---

# 事件日志

## 列出事件日志

获取带可选过滤功能的分页事件日志列表。

**列出所有事件日志：**
```bash
curl http://localhost:8002/osm/api/event-logs \
  -H "Authorization: Bearer ***"
```

**带分页：**
```bash
curl "http://localhost:8002/osm/api/event-logs?offset=0&limit=50" \
  -H "Authorization: Bearer ***"
```

**按项目空间过滤：**
```bash
curl "http://localhost:8002/osm/api/event-logs?workspace=example.com" \
  -H "Authorization: Bearer ***"
```

**按主题过滤：**
```bash
curl "http://localhost:8002/osm/api/event-logs?topic=run.completed" \
  -H "Authorization: Bearer ***"
```

**按运行 ID 过滤：**
```bash
curl "http://localhost:8002/osm/api/event-logs?run_id=abc12345" \
  -H "Authorization: Bearer ***"
```

**多条件过滤：**
```bash
curl "http://localhost:8002/osm/api/event-logs?workspace=example.com&processed=false&limit=100" \
  -H "Authorization: Bearer ***"
```

**查询参数：**

| 参数 | 类型 | 描述 |
|-----------|------|-------------|
| `topic` | string | 按事件主题过滤（例如 "run.started"、"run.completed"） |
| `name` | string | 按事件名称过滤 |
| `source` | string | 按事件来源过滤（例如 "executor"、"scheduler"、"api"） |
| `workspace` | string | 按项目空间名称过滤 |
| `run_id` | string | 按运行 ID 过滤 |
| `workflow_name` | string | 按工作流名称过滤 |
| `processed` | bool | 按处理状态过滤（"true" 或 "false"） |
| `offset` | int | 分页偏移量（默认：0） |
| `limit` | int | 最大返回记录数（默认：20，最大：10000） |

**响应：**
```json
{
  "data": [
    {
      "id": 1,
      "topic": "run.completed",
      "event_id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "subdomain-enum-completed",
      "source": "executor",
      "data_type": "scan",
      "data": "{\"scan_id\":\"abc12345\",\"target\":\"example.com\",\"duration_ms\":3600000,\"assets_found\":150,\"steps_completed\":10}",
      "workspace": "example.com",
      "run_id": "abc12345",
      "workflow_name": "subdomain-enum",
      "processed": true,
      "processed_at": "2025-01-15T10:30:00Z",
      "error": "",
      "created_at": "2025-01-15T09:30:00Z"
    },
    {
      "id": 2,
      "topic": "run.started",
      "event_id": "660e8400-e29b-41d4-a716-446655440001",
      "name": "port-scan-started",
      "source": "api",
      "data_type": "scan",
      "data": "{\"scan_id\":\"def67890\",\"target\":\"test.com\",\"params\":{\"ports\":\"top-1000\"}}",
      "workspace": "test.com",
      "run_id": "def67890",
      "workflow_name": "port-scan",
      "processed": true,
      "processed_at": "2025-01-15T11:00:00Z",
      "error": "",
      "created_at": "2025-01-15T11:00:00Z"
    },
    {
      "id": 3,
      "topic": "asset.discovered",
      "event_id": "770e8400-e29b-41d4-a716-446655440002",
      "name": "httpx-asset-found",
      "source": "executor",
      "data_type": "asset",
      "data": "{\"url\":\"https://api.example.com\",\"status_code\":200,\"title\":\"API Documentation\",\"tech\":[\"nginx\",\"nodejs\"]}",
      "workspace": "example.com",
      "run_id": "abc12345",
      "workflow_name": "subdomain-enum",
      "processed": true,
      "processed_at": "2025-01-15T10:15:00Z",
      "error": "",
      "created_at": "2025-01-15T10:15:00Z"
    },
    {
      "id": 4,
      "topic": "schedule.triggered",
      "event_id": "880e8400-e29b-41d4-a716-446655440003",
      "name": "daily-scan-triggered",
      "source": "scheduler",
      "data_type": "schedule",
      "data": "{\"schedule_id\":\"sch_1234567890\",\"trigger_type\":\"cron\",\"schedule\":\"0 2 * * *\"}",
      "workspace": "example.com",
      "run_id": "ghi11111",
      "workflow_name": "subdomain-enum",
      "processed": true,
      "processed_at": "2025-01-16T02:00:00Z",
      "error": "",
      "created_at": "2025-01-16T02:00:00Z"
    },
    {
      "id": 5,
      "topic": "run.failed",
      "event_id": "990e8400-e29b-41d4-a716-446655440004",
      "name": "nuclei-scan-failed",
      "source": "executor",
      "data_type": "scan",
      "data": "{\"scan_id\":\"jkl22222\",\"target\":\"unreachable.com\",\"error\":\"connection timeout\"}",
      "workspace": "unreachable.com",
      "run_id": "jkl22222",
      "workflow_name": "nuclei-scan",
      "processed": false,
      "processed_at": null,
      "error": "connection timeout after 5 retries",
      "created_at": "2025-01-15T14:00:00Z"
    },
    {
      "id": 6,
      "topic": "step.completed",
      "event_id": "aae8400-e29b-41d4-a716-446655440005",
      "name": "run-subfinder-completed",
      "source": "executor",
      "data_type": "step",
      "data": "{\"step_name\":\"run-subfinder\",\"duration_ms\":45000,\"output_lines\":150}",
      "workspace": "example.com",
      "run_id": "abc12345",
      "workflow_name": "subdomain-enum",
      "processed": true,
      "processed_at": "2025-01-15T10:01:45Z",
      "error": "",
      "created_at": "2025-01-15T10:01:45Z"
    }
  ],
  "pagination": {
    "total": 150,
    "offset": 0,
    "limit": 20
  }
}
```

**可用事件主题：**

| 主题 | 描述 |
|-------|-------------|
| `run.started` | 工作流执行已开始 |
| `run.completed` | 工作流执行成功完成 |
| `run.failed` | 工作流执行失败 |
| `asset.discovered` | 扫描期间发现新资产 |
| `asset.updated` | 现有资产信息已更新 |
| `webhook.received` | 收到外部 Webhook |
| `schedule.triggered` | 定时工作流已触发 |
| `step.completed` | 单个步骤已完成 |
| `step.failed` | 单个步骤已失败 |

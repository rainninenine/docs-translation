---
title: "调度"
description: "管理定时工作流触发器"
---

# 调度

## 列出调度

获取所有定时工作流的分页列表。

```bash
curl http://localhost:8002/osm/api/schedules \
  -H "Authorization: Bearer ***"
```

**带分页：**
```bash
curl "http://localhost:8002/osm/api/schedules?offset=0&limit=50" \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    {
      "id": "sch_1234567890",
      "name": "daily-scan",
      "workflow_name": "subdomain-enum",
      "workflow_path": "/home/user/osmedeus-base/workflows/flows/subdomain-enum.yaml",
      "trigger_name": "daily-scan-trigger",
      "trigger_type": "cron",
      "schedule": "0 2 * * *",
      "event_topic": "",
      "watch_path": "",
      "input_config": {
        "target": "example.com",
        "threads": "50"
      },
      "is_enabled": true,
      "last_run": "2025-01-15T02:00:00Z",
      "next_run": "2025-01-16T02:00:00Z",
      "run_count": 30,
      "created_at": "2025-01-01T00:00:00Z",
      "updated_at": "2025-01-15T02:00:00Z"
    },
    {
      "id": "sch_0987654321",
      "name": "weekly-full-recon",
      "workflow_name": "full-recon",
      "workflow_path": "/home/user/osmedeus-base/workflows/flows/full-recon.yaml",
      "trigger_name": "weekly-trigger",
      "trigger_type": "cron",
      "schedule": "0 0 * * 0",
      "event_topic": "",
      "watch_path": "",
      "input_config": {
        "target": "example.com",
        "threads": "100",
        "runner_type": "docker"
      },
      "is_enabled": true,
      "last_run": "2025-01-12T00:00:00Z",
      "next_run": "2025-01-19T00:00:00Z",
      "run_count": 5,
      "created_at": "2025-01-01T00:00:00Z",
      "updated_at": "2025-01-12T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 5,
    "offset": 0,
    "limit": 20
  }
}
```

---

## 创建调度

创建新的定时工作流执行。

```bash
curl -X POST http://localhost:8002/osm/api/schedules \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "daily-scan",
    "workflow_name": "subdomain-enum",
    "workflow_kind": "flow",
    "target": "example.com",
    "schedule": "0 2 * * *",
    "enabled": true
  }'
```

**带附加参数：**
```bash
curl -X POST http://localhost:8002/osm/api/schedules \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "weekly-full-scan",
    "workflow_name": "full-recon",
    "workflow_kind": "flow",
    "target": "example.com",
    "schedule": "0 0 * * 0",
    "enabled": true,
    "params": {
      "threads": "100"
    },
    "runner_type": "docker"
  }'
```

**响应：**
```json
{
  "message": "Schedule created",
  "data": {
    "id": "sch_1234567890",
    "name": "daily-scan",
    "workflow_name": "subdomain-enum",
    "schedule": "0 2 * * *",
    "is_enabled": true
  }
}
```

---

## 获取调度

获取特定调度的详细信息。

```bash
curl http://localhost:8002/osm/api/schedules/sch_1234567890 \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "id": "sch_1234567890",
  "name": "daily-scan",
  "workflow_name": "subdomain-enum",
  "workflow_path": "/home/user/osmedeus-base/workflows/flows/subdomain-enum.yaml",
  "trigger_name": "daily-scan-trigger",
  "trigger_type": "cron",
  "schedule": "0 2 * * *",
  "event_topic": "",
  "watch_path": "",
  "input_config": {
    "target": "example.com",
    "threads": "50"
  },
  "is_enabled": true,
  "last_run": "2025-01-15T02:00:00Z",
  "next_run": "2025-01-16T02:00:00Z",
  "run_count": 30,
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-15T02:00:00Z"
}
```

---

## 更新调度

更新现有调度。

```bash
curl -X PUT http://localhost:8002/osm/api/schedules/sch_1234567890 \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "updated-daily-scan",
    "schedule": "0 3 * * *"
  }'
```

**仅更新调度表达式：**
```bash
curl -X PUT http://localhost:8002/osm/api/schedules/sch_1234567890 \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "schedule": "0 4 * * *"
  }'
```

**响应：**
```json
{
  "message": "Schedule updated",
  "data": {
    "id": "sch_1234567890",
    "name": "updated-daily-scan",
    "schedule": "0 3 * * *"
  }
}
```

---

## 删除调度

删除一个调度。

```bash
curl -X DELETE http://localhost:8002/osm/api/schedules/sch_1234567890 \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Schedule deleted"
}
```

---

## 启用调度

启用一个已禁用的调度。

```bash
curl -X POST http://localhost:8002/osm/api/schedules/sch_1234567890/enable \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Schedule enabled"
}
```

---

## 禁用调度

禁用一个已启用的调度。

```bash
curl -X POST http://localhost:8002/osm/api/schedules/sch_1234567890/disable \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Schedule disabled"
}
```

---

## 触发调度

手动触发定时工作流执行。

```bash
curl -X POST http://localhost:8002/osm/api/schedules/sch_1234567890/trigger \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Schedule triggered",
  "schedule": "daily-scan",
  "workflow": "subdomain-enum"
}
```

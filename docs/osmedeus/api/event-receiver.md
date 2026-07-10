---
title: "事件接收器"
description: "事件触发的工作流管理"
---

# 事件接收器

这些端点仅在事件接收器启用时可用（调度器启用时默认启用）。

## 获取事件接收器状态

获取事件接收器的当前状态，包括已注册的触发器。

```bash
curl http://localhost:8002/osm/api/event-receiver/status \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "enabled": true,
  "running": true,
  "triggers": {
    "cron": 5,
    "event": 3,
    "watch": 2
  },
  "workflows_loaded": 10,
  "last_event_at": "2025-01-15T10:30:00Z"
}
```

---

## 列出事件接收器工作流

获取所有已在事件接收器中注册的工作流列表，包括它们的触发器。

```bash
curl http://localhost:8002/osm/api/event-receiver/workflows \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    {
      "name": "subdomain-enum",
      "kind": "flow",
      "triggers": [
        {
          "name": "daily-scan",
          "type": "cron",
          "schedule": "0 2 * * *",
          "enabled": true,
          "next_run": "2025-01-16T02:00:00Z"
        },
        {
          "name": "on-new-asset",
          "type": "event",
          "topic": "assets.new",
          "enabled": true
        }
      ]
    },
    {
      "name": "vuln-scan",
      "kind": "module",
      "triggers": [
        {
          "name": "watch-targets",
          "type": "watch",
          "path": "/data/targets/*.txt",
          "enabled": true
        }
      ]
    }
  ],
  "count": 2
}
```

---

## 发射事件

发射自定义事件以触发基于事件的工作流。

```bash
curl -X POST http://localhost:8002/osm/api/events/emit \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "topic": "assets.new",
    "data": {
      "url": "https://new-subdomain.example.com",
      "source": "external-scanner"
    }
  }'
```

**请求体：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `topic` | string | 是 | 要发射的事件主题（例如 `assets.new`、`custom.event`） |
| `data` | object | 否 | 传递给触发工作流的事件数据负载 |

**响应：**
```json
{
  "message": "Event emitted",
  "topic": "assets.new",
  "event_id": "evt_550e8400-e29b-41d4-a716-446655440000",
  "triggered_workflows": ["subdomain-enum", "asset-tracker"]
}
```

**错误响应（无匹配触发器）：**
```json
{
  "message": "Event emitted",
  "topic": "assets.new",
  "event_id": "evt_550e8400-e29b-41d4-a716-446655440000",
  "triggered_workflows": [],
  "note": "No workflows matched this event topic"
}
```

---

## 事件主题

工作流常用的事件主题：

| 主题 | 描述 |
|-------|-------------|
| `assets.new` | 发现新资产 |
| `assets.updated` | 资产信息已更新 |
| `vuln.found` | 发现新漏洞 |
| `run.completed` | 工作流运行已完成 |
| `run.failed` | 工作流运行已失败 |
| `schedule.triggered` | 定时触发器已触发 |

可以通过添加 `custom.` 前缀来使用自定义主题（例如 `custom.my-event`）。

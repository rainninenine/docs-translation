---
title: "步骤结果"
description: "查询步骤执行结果"
---

# 步骤结果

## 列出步骤结果

获取带可选过滤功能的分页步骤执行结果列表。

**列出所有步骤结果：**
```bash
curl http://localhost:8002/osm/api/step-results \
  -H "Authorization: Bearer ***"
```

**带分页：**
```bash
curl "http://localhost:8002/osm/api/step-results?offset=0&limit=50" \
  -H "Authorization: Bearer ***"
```

**按项目空间过滤：**
```bash
curl "http://localhost:8002/osm/api/step-results?workspace=example_com" \
  -H "Authorization: Bearer ***"
```

**按状态过滤：**
```bash
curl "http://localhost:8002/osm/api/step-results?status=completed" \
  -H "Authorization: Bearer ***"
```

**按步骤类型过滤：**
```bash
curl "http://localhost:8002/osm/api/step-results?step_type=bash" \
  -H "Authorization: Bearer ***"
```

**按运行 ID 过滤：**
```bash
curl "http://localhost:8002/osm/api/step-results?run_id=123" \
  -H "Authorization: Bearer ***"
```

**多条件过滤：**
```bash
curl "http://localhost:8002/osm/api/step-results?workspace=example_com&status=completed&limit=100" \
  -H "Authorization: Bearer ***"
```

**查询参数：**

| 参数 | 类型 | 描述 |
|-----------|------|-------------|
| `workspace` | string | 按项目空间名称过滤 |
| `status` | string | 按状态过滤（pending、running、completed、failed） |
| `step_type` | string | 按步骤类型过滤（bash、function、foreach、parallel-steps、remote-bash、http、llm） |
| `run_id` | int | 按运行 ID 过滤 |
| `offset` | int | 分页偏移量（默认：0） |
| `limit` | int | 最大返回记录数（默认：20，最大：10000） |

**响应：**
```json
{
  "data": [
    {
      "id": 1,
      "run_id": 123,
      "step_name": "run-subfinder",
      "step_type": "bash",
      "status": "completed",
      "command": "subfinder -d example.com -o {{Output}}/subdomains.txt",
      "output": "Found 150 subdomains",
      "error": "",
      "started_at": "2025-01-15T10:00:00Z",
      "completed_at": "2025-01-15T10:01:30Z",
      "duration_ms": 90000,
      "created_at": "2025-01-15T10:00:00Z",
      "updated_at": "2025-01-15T10:01:30Z"
    },
    {
      "id": 2,
      "run_id": 123,
      "step_name": "run-httpx",
      "step_type": "bash",
      "status": "completed",
      "command": "httpx -l {{Output}}/subdomains.txt -o {{Output}}/httpx.txt",
      "output": "Probed 150 hosts, 120 alive",
      "error": "",
      "started_at": "2025-01-15T10:01:30Z",
      "completed_at": "2025-01-15T10:03:00Z",
      "duration_ms": 90000,
      "created_at": "2025-01-15T10:01:30Z",
      "updated_at": "2025-01-15T10:03:00Z"
    },
    {
      "id": 3,
      "run_id": 124,
      "step_name": "process-results",
      "step_type": "function",
      "status": "completed",
      "command": "",
      "output": "true",
      "error": "",
      "started_at": "2025-01-15T11:00:00Z",
      "completed_at": "2025-01-15T11:00:01Z",
      "duration_ms": 1000,
      "created_at": "2025-01-15T11:00:00Z",
      "updated_at": "2025-01-15T11:00:01Z"
    }
  ],
  "pagination": {
    "total": 250,
    "offset": 0,
    "limit": 20
  }
}
```

**步骤状态值：**

| 状态 | 描述 |
|--------|-------------|
| `pending` | 步骤已排队但尚未开始 |
| `running` | 步骤正在执行 |
| `completed` | 步骤成功完成 |
| `failed` | 步骤失败并出现错误 |
| `skipped` | 步骤被跳过（pre_condition 未满足） |

**步骤类型：**

| 类型 | 描述 |
|------|-------------|
| `bash` | Shell 命令执行 |
| `function` | 工具函数执行 |
| `foreach` | 遍历输入项 |
| `parallel-steps` | 并行执行步骤 |
| `remote-bash` | 远程命令执行（Docker/SSH） |
| `http` | HTTP 请求步骤 |
| `llm` | LLM/AI 步骤 |

---
title: "产物"
description: "列出和下载工作流运行的输出产物"
---

# 产物

## 列出产物

获取跨项目空间的所有产物的分页列表。

```bash
curl http://localhost:8002/osm/api/artifacts \
  -H "Authorization: Bearer ***"
```

**带分页：**
```bash
curl "http://localhost:8002/osm/api/artifacts?offset=0&limit=50" \
  -H "Authorization: Bearer ***"
```

**按项目空间过滤：**
```bash
curl "http://localhost:8002/osm/api/artifacts?workspace=example.com" \
  -H "Authorization: Bearer ***"
```

**查询参数：**

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------|-------------|
| `workspace` | string | - | 按项目空间名称过滤 |
| `offset` | int | 0 | 分页偏移量 |
| `limit` | int | 20 | 最大返回记录数 |

**响应：**
```json
{
  "data": [
    {
      "id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
      "run_id": 1,
      "workspace": "example.com",
      "name": "subdomains.txt",
      "artifact_path": "/workspaces/example.com/subdomains.txt",
      "artifact_type": "output",
      "content_type": "txt",
      "size_bytes": 4523,
      "line_count": 150,
      "description": "Discovered subdomains",
      "created_at": "2025-01-15T10:01:45Z"
    },
    {
      "id": "d4e5f6a7-b8c9-0123-def0-234567890123",
      "run_id": 1,
      "workspace": "example.com",
      "name": "nuclei-results.json",
      "artifact_path": "/workspaces/example.com/nuclei-results.json",
      "artifact_type": "output",
      "content_type": "json",
      "size_bytes": 15234,
      "line_count": 45,
      "description": "Nuclei vulnerability scan results",
      "created_at": "2025-01-15T10:15:00Z"
    }
  ],
  "pagination": {
    "total": 100,
    "offset": 0,
    "limit": 20
  }
}
```

---

## 下载项目空间产物

将项目空间的所有产物下载为 zip 文件。

```bash
curl http://localhost:8002/osm/api/artifacts/example.com \
  -H "Authorization: Bearer ***" \
  --output example.com_artifacts.zip
```

**响应：**
- 成功时：返回包含所有产物的 zip 文件
- 响应头包括：
  - `Content-Disposition: attachment; filename=<workspace>_artifacts.zip`
  - `Content-Type: application/zip`

**错误响应（404）：**
```json
{
  "error": true,
  "message": "Workspace not found: example.com"
}
```

---

## 产物类型

| 类型 | 描述 |
|------|-------------|
| `report` | 从工作流的报告部分生成的报告 |
| `state_file` | 状态文件，如 run-state.json、run-execution.log |
| `output` | 步骤的通用输出文件 |
| `screenshot` | 扫描期间捕获的截图 |

## 内容类型

根据文件扩展名检测的内容类型：
- `json`、`jsonl`、`yaml`、`html`、`md`、`log`、`pdf`、`png`、`txt`、`zip`、`folder`、`unknown`

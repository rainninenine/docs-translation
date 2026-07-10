---
title: "状态快照"
description: "导出和导入项目空间快照"
---

# 状态快照

## 列出快照

获取快照目录中可用快照文件的列表。

```bash
curl http://localhost:8002/osm/api/snapshots \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    {
      "name": "example.com_1704067200.zip",
      "path": "/home/user/osmedeus-base/snapshot/example.com_1704067200.zip",
      "size": 15728640,
      "created_at": "2025-01-01T12:00:00Z"
    },
    {
      "name": "test.com_1704153600.zip",
      "path": "/home/user/osmedeus-base/snapshot/test.com_1704153600.zip",
      "size": 8388608,
      "created_at": "2025-01-02T12:00:00Z"
    }
  ],
  "count": 2,
  "path": "/home/user/osmedeus-base/snapshot"
}
```

---

## 导出项目空间快照

将项目空间导出为压缩的 zip 归档并下载。

```bash
curl -X POST http://localhost:8002/osm/api/snapshots/export \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{"workspace": "example.com"}' \
  --output example.com_snapshot.zip
```

**请求体：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `workspace` | string | 是 | 要导出的项目空间名称 |

**响应：**
- 成功时：返回 zip 文件作为二进制下载
- 响应头包括：
  - `Content-Disposition: attachment; filename=<workspace>_<timestamp>.zip`
  - `Content-Type: application/zip`
  - `X-Snapshot-Size: <size_in_bytes>`

**错误响应（404）：**
```json
{
  "error": true,
  "message": "Workspace not found: example.com"
}
```

---

## 导入项目空间快照

从上传的 zip 文件或 URL 导入项目空间。

**从文件上传导入：**
```bash
curl -X POST http://localhost:8002/osm/api/snapshots/import \
  -H "Authorization: Bearer ***" \
  -F "file=@example.com_1704067200.zip"
```

**从 URL 导入：**
```bash
curl -X POST http://localhost:8002/osm/api/snapshots/import \
  -H "Authorization: Bearer ***" \
  -F "url=https://example.com/snapshots/workspace.zip"
```

**带强制覆盖导入：**
```bash
curl -X POST http://localhost:8002/osm/api/snapshots/import \
  -H "Authorization: Bearer ***" \
  -F "file=@example.com_snapshot.zip" \
  -F "force=true"
```

**仅导入文件（跳过数据库）：**
```bash
curl -X POST http://localhost:8002/osm/api/snapshots/import \
  -H "Authorization: Bearer ***" \
  -F "file=@example.com_snapshot.zip" \
  -F "skip_db=true"
```

**表单参数：**

| 参数 | 类型 | 必填 | 描述 |
|-----------|------|----------|-------------|
| `file` | file | 否* | 要导入的快照 zip 文件 |
| `url` | string | 否* | 要下载并导入的快照 URL |
| `force` | bool | 否 | 覆盖现有项目空间（默认：false） |
| `skip_db` | bool | 否 | 跳过数据库导入，仅解压文件（默认：false） |

*`file` 或 `url` 中必须指定一个。

**响应（200）：**
```json
{
  "message": "Workspace imported successfully",
  "workspace": "example.com",
  "local_path": "/home/user/workspaces-osmedeus/example.com",
  "data_source": "imported",
  "files_count": 1523,
  "warning": "Imported workspace database state may be unstable. Only import from trusted sources."
}
```

**错误响应（400）：**
```json
{
  "error": true,
  "message": "Either file or url is required"
}
```

**错误响应（500 - 项目空间已存在）：**
```json
{
  "error": true,
  "message": "Failed to import snapshot: workspace already exists: /home/user/workspaces-osmedeus/example.com (use --force to overwrite)"
}
```

---

## 删除快照

按名称删除快照文件。

```bash
curl -X DELETE http://localhost:8002/osm/api/snapshots/example.com_1704067200.zip \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Snapshot deleted successfully",
  "name": "example.com_1704067200.zip"
}
```

**错误响应（404）：**
```json
{
  "error": true,
  "message": "Snapshot not found: example.com_1704067200.zip"
}
```

---

## 旧版端点

旧版快照下载端点仍可用于向后兼容：

```bash
curl http://localhost:8002/osm/api/snapshot-download/example.com \
  -H "Authorization: Bearer ***" \
  --output snapshot.zip
```

---

## 数据来源值

导入项目空间时，其 `data_source` 字段被设置为指示其创建方式：

| 值 | 描述 |
|-------|-------------|
| `local` | 通过扫描在本地创建（默认） |
| `cloud` | 从云存储同步 |
| `imported` | 从快照文件导入 |

---

## 安全注意事项

**警告：** 仅从可信来源导入快照！

导入的项目空间数据可能包含：
- 可能与现有数据冲突的数据库记录
- 引用外部资源的文件路径
- 可能不兼容的配置

导入的项目空间数据库状态可能不稳定。如果只需要文件而不需要数据库导入，请使用 `skip_db=true` 参数。

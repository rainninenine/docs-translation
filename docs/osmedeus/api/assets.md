---
title: "资产"
description: "查询已发现的资产和资产差异快照"
---

# 资产

## 列出资产

获取带可选项目空间过滤的分页资产列表。

**列出所有资产：**
```bash
curl http://localhost:8002/osm/api/assets \
  -H "Authorization: Bearer ***"
```

**带分页列出资产：**
```bash
curl "http://localhost:8002/osm/api/assets?offset=0&limit=100" \
  -H "Authorization: Bearer ***"
```

**按项目空间过滤：**
```bash
curl "http://localhost:8002/osm/api/assets?workspace=example.com" \
  -H "Authorization: Bearer ***"
```

**组合项目空间过滤与分页：**
```bash
curl "http://localhost:8002/osm/api/assets?workspace=example.com&offset=50&limit=25" \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    {
      "id": 1,
      "workspace": "example.com",
      "asset_value": "api.example.com",
      "url": "https://api.example.com",
      "input": "api.example.com",
      "scheme": "https",
      "method": "GET",
      "path": "/",
      "status_code": 200,
      "content_type": "application/json",
      "content_length": 4523,
      "title": "API Documentation",
      "words": 523,
      "lines": 89,
      "host_ip": "93.184.216.34",
      "a": ["93.184.216.34", "93.184.216.35"],
      "tls": "TLS 1.3",
      "asset_type": "web",
      "tech": ["nginx/1.21.0", "nodejs", "express"],
      "time": "245ms",
      "remarks": ["production"],
      "source": "httpx",
      "created_at": "2025-01-15T10:30:00Z",
      "updated_at": "2025-01-15T10:30:00Z"
    },
    {
      "id": 2,
      "workspace": "example.com",
      "asset_value": "admin.example.com",
      "url": "https://admin.example.com",
      "input": "admin.example.com",
      "scheme": "https",
      "method": "GET",
      "path": "/login",
      "status_code": 401,
      "content_type": "text/html",
      "content_length": 2156,
      "title": "Admin Login - Example Corp",
      "words": 156,
      "lines": 45,
      "host_ip": "93.184.216.36",
      "a": ["93.184.216.36"],
      "tls": "TLS 1.2",
      "asset_type": "web",
      "tech": ["nginx/1.20.0", "php/8.1", "wordpress"],
      "time": "312ms",
      "remarks": ["admin-panel"],
      "source": "httpx",
      "created_at": "2025-01-15T10:31:00Z",
      "updated_at": "2025-01-15T10:31:00Z"
    }
  ],
  "pagination": {
    "total": 500,
    "offset": 0,
    "limit": 20
  }
}
```

**资产字段参考：**

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `id` | int | 唯一资产标识符 |
| `workspace` | string | 项目空间/扫描目标名称 |
| `asset_value` | string | 主要资产标识符（主机名/子域名） |
| `url` | string | 资产的完整 URL |
| `input` | string | 原始输入值 |
| `scheme` | string | 协议方案（http、https） |
| `method` | string | 使用的 HTTP 方法 |
| `path` | string | URL 路径 |
| `status_code` | int | HTTP 响应状态码 |
| `content_type` | string | 响应内容类型 |
| `content_length` | int | 响应体大小（字节） |
| `title` | string | HTML 页面标题 |
| `words` | int | 响应中的字数 |
| `lines` | int | 响应中的行数 |
| `host_ip` | string | 解析的 IP 地址 |
| `a` | array | DNS A 记录 |
| `tls` | string | TLS 版本信息 |
| `asset_type` | string | 资产类型分类 |
| `tech` | array | 检测到的技术栈 |
| `time` | string | 响应时间 |
| `remarks` | array | 自定义标签/备注 |
| `language` | string | 仓库/源代码语言 |
| `size` | int | 仓库/文件大小（字节） |
| `loc` | int | 代码行数（针对仓库资产） |
| `blob_content` | string | 大内容（原始文件、密钥等） |
| `source` | string | 发现来源（httpx、nuclei 等） |
| `created_at` | timestamp | 创建时间戳 |
| `updated_at` | timestamp | 最后更新时间戳 |

---

## 获取资产统计

获取所有资产中技术栈、来源、备注和资产类型的唯一值。

**获取所有资产统计：**
```bash
curl http://localhost:8002/osm/api/asset-stats \
  -H "Authorization: Bearer ***"
```

**按项目空间过滤：**
```bash
curl "http://localhost:8002/osm/api/asset-stats?workspace=example.com" \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": {
    "technologies": ["nginx/1.21.0", "nodejs", "php/8.1", "wordpress"],
    "sources": ["httpx", "nuclei"],
    "remarks": ["admin-panel", "production"],
    "asset_types": ["http", "dns", "subdomain"]
  }
}
```

**查询参数：**

| 参数 | 类型 | 必填 | 描述 |
|-----------|------|----------|-------------|
| `workspace` | string | 否 | 按项目空间名称过滤统计 |

**响应字段：**

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `technologies` | array | 所有匹配资产中检测到的唯一技术栈 |
| `sources` | array | 唯一的发现来源（httpx、nuclei 等） |
| `remarks` | array | 唯一的自定义标签/备注 |
| `asset_types` | array | 唯一的资产类型分类（http、dns、subdomain、ip 等） |

---

## 列出资产差异快照

获取存储的资产差异快照的分页列表。这些快照捕获资产随时间的变化。

**列出所有资产差异快照：**
```bash
curl http://localhost:8002/osm/api/assets/diffs \
  -H "Authorization: Bearer ***"
```

**带分页列出：**
```bash
curl "http://localhost:8002/osm/api/assets/diffs?offset=0&limit=50" \
  -H "Authorization: Bearer ***"
```

**按项目空间过滤：**
```bash
curl "http://localhost:8002/osm/api/assets/diffs?workspace=example.com" \
  -H "Authorization: Bearer ***"
```

**组合项目空间过滤与分页：**
```bash
curl "http://localhost:8002/osm/api/assets/diffs?workspace=example.com&offset=0&limit=25" \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    {
      "id": 1,
      "workspace_name": "example.com",
      "from_time": "2025-01-14T00:00:00Z",
      "to_time": "2025-01-15T00:00:00Z",
      "total_added": 15,
      "total_removed": 3,
      "total_changed": 7,
      "diff_data": "{\"added\":[...],\"removed\":[...],\"changed\":[...]}",
      "created_at": "2025-01-15T10:30:00Z"
    },
    {
      "id": 2,
      "workspace_name": "example.com",
      "from_time": "2025-01-15T00:00:00Z",
      "to_time": "2025-01-16T00:00:00Z",
      "total_added": 8,
      "total_removed": 1,
      "total_changed": 12,
      "diff_data": "{\"added\":[...],\"removed\":[...],\"changed\":[...]}",
      "created_at": "2025-01-16T10:30:00Z"
    }
  ],
  "pagination": {
    "total": 30,
    "offset": 0,
    "limit": 20
  }
}
```

**资产差异快照字段参考：**

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `id` | int | 唯一快照标识符 |
| `workspace_name` | string | 此差异对应的项目空间名称 |
| `from_time` | timestamp | 差异周期的开始时间 |
| `to_time` | timestamp | 差异周期的结束时间 |
| `total_added` | int | 新增资产数量 |
| `total_removed` | int | 移除资产数量 |
| `total_changed` | int | 发生变化的资产数量 |
| `diff_data` | string | JSON 序列化的差异数据，包含新增、移除和变化的资产 |
| `created_at` | timestamp | 快照创建时间 |

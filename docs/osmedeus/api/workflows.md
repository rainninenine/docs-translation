---
title: "工作流"
description: "列出、查看和刷新工作流"
---

# 工作流

## 列出所有工作流

获取所有可用工作流的分页列表，支持过滤。

```bash
curl http://localhost:8002/osm/api/workflows \
  -H "Authorization: Bearer ***"
```

**查询参数：**

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------|-------------|
| `source` | string | `db` | 数据来源：`db`（数据库）或 `filesystem`（直接文件扫描） |
| `tags` | string | - | 逗号分隔的标签列表，用于过滤 |
| `kind` | string | - | 按工作流类型过滤：`flow`、`module` |
| `search` | string | - | 在工作流名称和描述中搜索 |
| `offset` | int | 0 | 分页偏移量 |
| `limit` | int | 50 | 最大返回记录数 |

**示例：**

```bash
# 按标签过滤
curl "http://localhost:8002/osm/api/workflows?tags=recon,subdomain" \
  -H "Authorization: Bearer ***"

# 按类型过滤（module）
curl "http://localhost:8002/osm/api/workflows?kind=module" \
  -H "Authorization: Bearer ***"

# 搜索工作流
curl "http://localhost:8002/osm/api/workflows?search=enum" \
  -H "Authorization: Bearer ***"

# 直接从文件系统加载（绕过数据库）
curl "http://localhost:8002/osm/api/workflows?source=filesystem" \
  -H "Authorization: Bearer ***"

# 分页
curl "http://localhost:8002/osm/api/workflows?offset=10&limit=20" \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    {
      "name": "subdomain-enum",
      "kind": "flow",
      "description": "Comprehensive subdomain enumeration and probing workflow",
      "tags": ["recon", "subdomain", "httpx"],
      "file_path": "/home/user/osmedeus-base/workflows/flows/subdomain-enum.yaml",
      "params": [
        {"name": "target", "required": true, "default": "", "generator": ""},
        {"name": "threads", "required": false, "default": "50", "generator": ""},
        {"name": "timeout", "required": false, "default": "30", "generator": ""},
        {"name": "wordlist", "required": false, "default": "", "generator": "default_wordlist"}
      ],
      "required_params": ["target"],
      "step_count": 8,
      "module_count": 3,
      "checksum": "sha256:abc123...",
      "indexed_at": "2025-01-15T08:00:00Z"
    },
    {
      "name": "port-scan",
      "kind": "module",
      "description": "Port scanning module using nmap and masscan",
      "tags": ["recon", "portscan", "nmap"],
      "file_path": "/home/user/osmedeus-base/workflows/modules/port-scan.yaml",
      "params": [
        {"name": "target", "required": true, "default": "", "generator": ""},
        {"name": "ports", "required": false, "default": "top-1000", "generator": ""},
        {"name": "rate", "required": false, "default": "1000", "generator": ""}
      ],
      "required_params": ["target"],
      "step_count": 4,
      "module_count": 0,
      "checksum": "sha256:def456...",
      "indexed_at": "2025-01-15T08:00:00Z"
    }
  ],
  "pagination": {
    "total": 25,
    "offset": 0,
    "limit": 50
  }
}
```

---

## 获取工作流标签

获取已索引工作流的所有唯一标签。

```bash
curl http://localhost:8002/osm/api/workflows/tags \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "tags": ["recon", "subdomain", "portscan", "vulnerability", "nuclei"],
  "count": 5
}
```

---

## 刷新工作流索引

从文件系统重新索引所有工作流到数据库。添加或修改工作流文件后使用此功能。

```bash
curl -X POST http://localhost:8002/osm/api/workflows/refresh \
  -H "Authorization: Bearer ***"
```

**查询参数：**

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------|-------------|
| `force` | bool | false | 强制重新索引所有工作流，忽略校验和 |

**强制重新索引所有：**
```bash
curl -X POST "http://localhost:8002/osm/api/workflows/refresh?force=true" \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Workflows indexed successfully",
  "added": 5,
  "updated": 2,
  "removed": 1,
  "errors": []
}
```

---

## 获取工作流详情

获取工作流内容。默认返回原始 YAML，或返回包含完整解析详情的 JSON。

```bash
# 获取原始 YAML 内容（默认）
curl http://localhost:8002/osm/api/workflows/subdomain-enum \
  -H "Authorization: Bearer ***"
```

```bash
# 以 JSON 格式获取工作流详情
curl "http://localhost:8002/osm/api/workflows/subdomain-enum?json=true" \
  -H "Authorization: Bearer ***"
```

**查询参数：**

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------|-------------|
| `json` | bool | false | 返回包含解析详情的 JSON 而非原始 YAML |

**响应（YAML - 默认）：**
```yaml
name: subdomain-enum
kind: flow
description: Subdomain enumeration
params:
  - name: target
    required: true
steps:
  - name: run-subfinder
    command: subfinder -d {{target}}
...
```

**响应（JSON，使用 `?json=true`）：**
```json
{
  "name": "subdomain-enum",
  "kind": "flow",
  "description": "Comprehensive subdomain enumeration and probing workflow",
  "file_path": "/home/user/osmedeus-base/workflows/flows/subdomain-enum.yaml",
  "params": [
    {"name": "target", "required": true, "default": "", "generator": ""},
    {"name": "threads", "required": false, "default": "50", "generator": ""},
    {"name": "output_dir", "required": false, "default": "{{Workspace}}", "generator": "workspace_path"},
    {"name": "wordlist", "required": false, "default": "", "generator": "default_wordlist"}
  ],
  "steps": [
    {
      "index": 0,
      "name": "run-subfinder",
      "type": "bash",
      "command": "subfinder -d {{target}} -t {{threads}} -o {{output_dir}}/subdomains-subfinder.txt",
      "timeout": "30m",
      "pre_condition": "",
      "exports": {"subfinder_output": "{{output_dir}}/subdomains-subfinder.txt"}
    },
    {
      "index": 1,
      "name": "run-amass",
      "type": "bash",
      "command": "amass enum -passive -d {{target}} -o {{output_dir}}/subdomains-amass.txt",
      "timeout": "60m",
      "pre_condition": "commandExists('amass')",
      "exports": {"amass_output": "{{output_dir}}/subdomains-amass.txt"}
    },
    {
      "index": 2,
      "name": "merge-subdomains",
      "type": "function",
      "command": "mergeFiles('{{output_dir}}/subdomains-*.txt', '{{output_dir}}/all-subdomains.txt')",
      "timeout": "",
      "pre_condition": "",
      "exports": {"all_subdomains": "{{output_dir}}/all-subdomains.txt"}
    },
    {
      "index": 3,
      "name": "run-httpx",
      "type": "bash",
      "command": "httpx -l {{all_subdomains}} -t {{threads}} -o {{output_dir}}/alive.txt -json -o {{output_dir}}/httpx.json",
      "timeout": "60m",
      "pre_condition": "fileLength('{{all_subdomains}}') > 0",
      "exports": {"alive_hosts": "{{output_dir}}/alive.txt", "httpx_json": "{{output_dir}}/httpx.json"}
    }
  ],
  "modules": [
    {"index": 0, "name": "port-scan", "path": "modules/port-scan.yaml", "depends_on": [], "condition": ""},
    {"index": 1, "name": "nuclei-scan", "path": "modules/nuclei-scan.yaml", "depends_on": ["port-scan"], "condition": "fileLength('{{alive_hosts}}') > 0"},
    {"index": 2, "name": "screenshot", "path": "modules/screenshot.yaml", "depends_on": ["port-scan"], "condition": ""}
  ],
  "triggers": [
    {"name": "daily-scan", "on": "cron", "schedule": "0 2 * * *", "enabled": true},
    {"name": "on-new-asset", "on": "event", "topic": "asset.discovered", "enabled": false}
  ],
  "dependencies": {
    "commands": ["subfinder", "amass", "httpx", "nuclei", "nmap"],
    "files": ["{{wordlist}}"]
  }
}
```

**步骤依赖：**

步骤可以使用 `depends_on` 字段依赖其他步骤：
```json
{
  "steps": [
    {
      "index": 0,
      "name": "step-a",
      "type": "bash",
      "command": "echo A"
    },
    {
      "index": 1,
      "name": "step-b",
      "type": "bash",
      "command": "echo B",
      "depends_on": ["step-a"]
    },
    {
      "index": 2,
      "name": "step-c",
      "type": "bash",
      "command": "echo C",
      "depends_on": ["step-a", "step-b"]
    }
  ]
}
```

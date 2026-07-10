---
title: "漏洞"
description: "查看和管理漏洞记录"
---

# 漏洞

## 列出漏洞

获取带可选过滤功能的分页漏洞列表，可按项目空间、严重级别、置信度或资产值过滤。

**列出所有漏洞：**
```bash
curl http://localhost:8002/osm/api/vulnerabilities \
  -H "Authorization: Bearer ***"
```

**带分页列出漏洞：**
```bash
curl "http://localhost:8002/osm/api/vulnerabilities?offset=0&limit=100" \
  -H "Authorization: Bearer ***"
```

**按项目空间过滤：**
```bash
curl "http://localhost:8002/osm/api/vulnerabilities?workspace=example.com" \
  -H "Authorization: Bearer ***"
```

**按严重级别过滤：**
```bash
curl "http://localhost:8002/osm/api/vulnerabilities?severity=critical" \
  -H "Authorization: Bearer ***"
```

**按置信度过滤：**
```bash
curl "http://localhost:8002/osm/api/vulnerabilities?confidence=Certain" \
  -H "Authorization: Bearer ***"
```

**按资产值过滤（部分匹配）：**
```bash
curl "http://localhost:8002/osm/api/vulnerabilities?asset_value=api.example" \
  -H "Authorization: Bearer ***"
```

**组合过滤条件：**
```bash
curl "http://localhost:8002/osm/api/vulnerabilities?workspace=example.com&severity=high&offset=0&limit=50" \
  -H "Authorization: Bearer ***"
```

**查询参数：**

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------|-------------|
| `workspace` | string | - | 按项目空间名称过滤 |
| `severity` | string | - | 按严重级别过滤（critical、high、medium、low、info） |
| `confidence` | string | - | 按置信度过滤（Certain、Firm、Tentative、Manual Review Required） |
| `asset_value` | string | - | 按资产值过滤（部分匹配） |
| `offset` | int | 0 | 要跳过的记录数 |
| `limit` | int | 20 | 最大返回记录数（最大 10000） |

**响应：**
```json
{
  "data": [
    {
      "id": 1,
      "workspace": "example.com",
      "vuln_info": "CVE-2024-1234",
      "vuln_title": "SQL Injection in Login Form",
      "vuln_desc": "The login form is vulnerable to SQL injection via the username parameter.",
      "vuln_poc": "username=' OR '1'='1' --&password=test",
      "severity": "critical",
      "confidence": "Certain",
      "asset_type": "web",
      "asset_value": "https://example.com/login",
      "tags": ["sqli", "owasp-top10", "authentication"],
      "detail_http_request": "POST /login HTTP/1.1\nHost: example.com\n...",
      "detail_http_response": "HTTP/1.1 200 OK\n...",
      "raw_vuln_json": "{\"template\":\"sqli-login.yaml\",...}",
      "created_at": "2025-01-15T10:30:00Z",
      "updated_at": "2025-01-15T10:30:00Z"
    },
    {
      "id": 2,
      "workspace": "example.com",
      "vuln_info": "CVE-2024-5678",
      "vuln_title": "Cross-Site Scripting (XSS) in Search",
      "vuln_desc": "Reflected XSS vulnerability in the search functionality.",
      "vuln_poc": "<script>alert('XSS')</script>",
      "severity": "high",
      "confidence": "Firm",
      "asset_type": "web",
      "asset_value": "https://example.com/search",
      "tags": ["xss", "owasp-top10"],
      "detail_http_request": "GET /search?q=<script>alert(1)</script> HTTP/1.1\n...",
      "detail_http_response": "HTTP/1.1 200 OK\n...",
      "raw_vuln_json": "{\"template\":\"xss-reflected.yaml\",...}",
      "created_at": "2025-01-15T10:31:00Z",
      "updated_at": "2025-01-15T10:31:00Z"
    }
  ],
  "pagination": {
    "total": 15,
    "offset": 0,
    "limit": 20
  }
}
```

---

## 获取漏洞摘要

获取按严重级别分组的漏洞摘要，可选按项目空间过滤。

**获取所有项目空间的摘要：**
```bash
curl http://localhost:8002/osm/api/vulnerabilities/summary \
  -H "Authorization: Bearer ***"
```

**获取特定项目空间的摘要：**
```bash
curl "http://localhost:8002/osm/api/vulnerabilities/summary?workspace=example.com" \
  -H "Authorization: Bearer ***"
```

**查询参数：**

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------|-------------|
| `workspace` | string | - | 按项目空间名称过滤 |

**响应：**
```json
{
  "data": {
    "by_severity": {
      "critical": 2,
      "high": 5,
      "medium": 8,
      "low": 12,
      "info": 3
    },
    "total": 30,
    "workspace": "example.com"
  }
}
```

---

## 按 ID 获取漏洞

通过 ID 检索单个漏洞。

```bash
curl http://localhost:8002/osm/api/vulnerabilities/1 \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": {
    "id": 1,
    "workspace": "example.com",
    "vuln_info": "CVE-2024-1234",
    "vuln_title": "SQL Injection in Login Form",
    "vuln_desc": "The login form is vulnerable to SQL injection via the username parameter.",
    "vuln_poc": "username=' OR '1'='1' --&password=test",
    "severity": "critical",
    "confidence": "Certain",
    "asset_type": "web",
    "asset_value": "https://example.com/login",
    "tags": ["sqli", "owasp-top10", "authentication"],
    "detail_http_request": "POST /login HTTP/1.1\nHost: example.com\n...",
    "detail_http_response": "HTTP/1.1 200 OK\n...",
    "raw_vuln_json": "{\"template\":\"sqli-login.yaml\",...}",
    "created_at": "2025-01-15T10:30:00Z",
    "updated_at": "2025-01-15T10:30:00Z"
  }
}
```

**错误响应（404）：**
```json
{
  "error": true,
  "message": "Vulnerability not found"
}
```

---

## 创建漏洞

创建新的漏洞记录。

```bash
curl -X POST http://localhost:8002/osm/api/vulnerabilities \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "workspace": "example.com",
    "vuln_info": "CVE-2024-9999",
    "vuln_title": "Remote Code Execution",
    "vuln_desc": "Critical RCE vulnerability in admin panel.",
    "vuln_poc": "curl -X POST /admin/exec -d \"cmd=id\"",
    "severity": "critical",
    "asset_type": "web",
    "asset_value": "https://example.com/admin",
    "tags": ["rce", "critical", "admin"],
    "detail_http_request": "POST /admin/exec HTTP/1.1\n...",
    "detail_http_response": "HTTP/1.1 200 OK\nuid=0(root)...",
    "raw_vuln_json": "{\"template\":\"rce-admin.yaml\"}"
  }'
```

**请求体：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `workspace` | string | 是 | 项目空间/目标名称 |
| `vuln_info` | string | 否 | CVE 或漏洞标识符 |
| `vuln_title` | string | 否 | 漏洞简短标题 |
| `vuln_desc` | string | 否 | 详细描述 |
| `vuln_poc` | string | 否 | 概念验证 |
| `severity` | string | 否 | 严重级别（critical、high、medium、low、info） |
| `confidence` | string | 否 | 置信度（Certain、Firm、Tentative、Manual Review Required） |
| `asset_type` | string | 否 | 资产类型（web、api、network 等） |
| `asset_value` | string | 否 | 受影响的资产 URL 或标识符 |
| `tags` | array | 否 | 分类标签 |
| `detail_http_request` | string | 否 | 原始 HTTP 请求 |
| `detail_http_response` | string | 否 | 原始 HTTP 响应 |
| `raw_vuln_json` | string | 否 | 来自扫描器（nuclei 等）的原始 JSON |

**响应（201 Created）：**
```json
{
  "data": {
    "id": 15,
    "workspace": "example.com",
    "vuln_info": "CVE-2024-9999",
    "vuln_title": "Remote Code Execution",
    "vuln_desc": "Critical RCE vulnerability in admin panel.",
    "vuln_poc": "curl -X POST /admin/exec -d \"cmd=id\"",
    "severity": "critical",
    "confidence": "Certain",
    "asset_type": "web",
    "asset_value": "https://example.com/admin",
    "tags": ["rce", "critical", "admin"],
    "detail_http_request": "POST /admin/exec HTTP/1.1\n...",
    "detail_http_response": "HTTP/1.1 200 OK\nuid=0(root)...",
    "raw_vuln_json": "{\"template\":\"rce-admin.yaml\"}",
    "created_at": "2025-01-15T14:25:00Z",
    "updated_at": "2025-01-15T14:25:00Z"
  },
  "message": "Vulnerability created successfully"
}
```

**错误响应（400）：**
```json
{
  "error": true,
  "message": "Workspace is required"
}
```

---

## 删除漏洞

按 ID 删除漏洞。

```bash
curl -X DELETE http://localhost:8002/osm/api/vulnerabilities/15 \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Vulnerability deleted successfully"
}
```

**错误响应（404）：**
```json
{
  "error": true,
  "message": "Vulnerability not found"
}
```

---

## 漏洞字段参考

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `id` | int | 唯一漏洞标识符 |
| `workspace` | string | 项目空间/扫描目标名称 |
| `vuln_info` | string | CVE 或漏洞标识符 |
| `vuln_title` | string | 简短描述性标题 |
| `vuln_desc` | string | 详细漏洞描述 |
| `vuln_poc` | string | 概念验证利用 |
| `severity` | string | 严重级别（critical、high、medium、low、info） |
| `confidence` | string | 置信度（Certain、Firm、Tentative、Manual Review Required） |
| `asset_type` | string | 受影响资产类型 |
| `asset_value` | string | 受影响资产 URL 或标识符 |
| `tags` | array | 分类标签 |
| `detail_http_request` | string | 触发漏洞的原始 HTTP 请求 |
| `detail_http_response` | string | 来自易受攻击端点的原始 HTTP 响应 |
| `raw_vuln_json` | string | 来自漏洞扫描器的原始 JSON 输出 |
| `created_at` | timestamp | 记录创建时间戳 |
| `updated_at` | timestamp | 最后更新时间戳 |

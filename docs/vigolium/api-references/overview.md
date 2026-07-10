# Vigolium API Reference — Overview

Base URL: `http://localhost:9002` (default)

## Starting the Server

```bash
# No authentication (development)
vigolium server -A

# With API key
export VIGOLIUM_API_KEY="my-secret-key"
vigolium server -A

# Custom host/port
vigolium server -A --host 127.0.0.1 --service-port 8080
```

## Authentication

All endpoints registered after the auth middleware require a Bearer token when the server is started with the `VIGOLIUM_API_KEY` environment variable or `server.auth_api_key` config. 这包括 `GET /`, `GET /health`, `GET /server-info`, and all `/api/*` routes.

```bash
curl -H "Authorization: Bearer my-secret-key" http://localhost:9002/api/stats
```

Public endpoints (no auth required): `GET /swagger/*`, `GET /metrics`.

## Project Scoping

All API operations are scoped to a project 通过 the `X-Project-UUID` request header. If the header is omitted, 默认 project (`00000000-0000-0000-0000-000000000001`) is used.

```bash
# Scope requests to a specific project
curl -H "Authorization: Bearer my-secret-key" \
     -H "X-Project-UUID: a1b2c3d4-..." \
     http://localhost:9002/api/findings
```

This applies to all data endpoints: ingestion, findings, HTTP records, stats, scans, source repos, and OAST（带外交互） interactions. See [Projects](../projects.md) for the full multi-tenancy reference.

---

## GET / — App Info

返回 basic application metadata.

```bash
curl -s http://localhost:9002/ | jq .
```

```json
{
  "name": "vigolium",
  "version": "v0.0.1-alpha",
  "author": "@j3ssie",
  "docs": "https://docs.vigolium.com",
  "build_time": "2026-02-16T15:22:43Z",
  "commit": "67bdce4"
}
```

---

## GET /api/diagnostics — System Readiness Check

返回 a diagnostic report checking database, agent backend, third-party tools, queue health, and directory configuration. See [Diagnostics](diagnostics.md) for full details.

```bash
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:9002/api/diagnostics | jq .status
```

```json
"ready"
```

---

## GET /health — Health Check

返回 server health status.

```bash
curl -s http://localhost:9002/health | jq .
```

```json
{
  "status": "healthy",
  "timestamp": "2026-02-16T15:30:00Z"
}
```

---

## GET /server-info — Server Info

返回 detailed server information including uptime, database driver, queue depth, and record/finding totals.

```bash
curl -s http://localhost:9002/server-info | jq .
```

```json
{
  "name": "vigolium",
  "version": "v0.0.1-alpha",
  "author": "@j3ssie",
  "docs": "https://docs.vigolium.com",
  "build_time": "2026-02-16T15:22:43Z",
  "commit": "67bdce4",
  "uptime": "5m32s",
  "service_addr": "0.0.0.0:9002",
  "proxy_addr": "",
  "queue_depth": 0,
  "total_records": 1234,
  "total_findings": 42,
  "license": "community-demo",
  "demo_only": true,
  "view_only": false
}
```

`license`, `demo_only`, and `view_only` are omitted when unset/false. Configure `license` under `server.license` in `vigolium-configs.yaml`; `demo_only` / `view_only` reflect the `--demo-only` / `--view-only` server flags.

---

## GET /swagger/* — Swagger UI

Interactive API documentation. Open in a browser.

```
http://localhost:9002/swagger/
```

The raw OpenAPI 3.0 spec is available at:

```bash
curl -s http://localhost:9002/swagger/doc.json | jq .info
```

---

## GET /metrics — Prometheus Metrics

返回 Prometheus-formatted metrics. No authentication required. Only available when the server is started with `--enable-metrics`.

```bash
curl -s http://localhost:9002/metrics
```

---

## CORS

CORS can be enabled 通过 the `cors_allowed_origins` server config:

| Value              | Behavior                                              |
|--------------------|-------------------------------------------------------|
| `*`                | Allow all origins                                     |
| `reflect-origin`   | Reflect the request's `Origin` header (allows credentials) |
| `origin1,origin2`  | Allow specific origins (comma-separated, allows credentials) |
| *(empty/omitted)*  | CORS disabled                                         |

Allowed methods: `GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`. Allowed headers: `Content-Type`, `Authorization`.

---

## Error Responses

All errors follow a consistent format:

```json
{
  "error": "error message",
  "code": 400,
  "details": "optional additional details"
}
```

**Common error codes:**

| Code | Meaning                                      |
|------|----------------------------------------------|
| 400  | Bad request (invalid JSON, missing fields)   |
| 401  | Unauthorized (missing or invalid Bearer token) |
| 404  | Not found (e.g. agent run ID not found)      |
| 409  | Conflict (scan or agent already running)     |
| 500  | Internal server error                         |
| 503  | 数据库 not available                        |

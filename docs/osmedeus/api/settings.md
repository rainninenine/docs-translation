---
title: "设置"
description: "服务器配置管理"
---

# 设置

管理服务器配置设置。

## 获取 YAML 配置

获取完整的 YAML 配置文件，敏感字段将被遮蔽。包含 `_key`、`secret`、`password`、`username` 或 `_token` 的字段将被替换为 `[REDACTED]`。

```bash
curl http://localhost:8002/osm/api/settings/yaml \
  -H "Authorization: Bearer ***"
```

**响应：** (text/yaml)
```yaml
# =============================================================================
# Osmedeus Configuration File
# =============================================================================
base_folder: ~/osmedeus-base

environments:
  binaries_path: "{{base_folder}}/binaries"
  # ... 更多配置

server:
  host: "0.0.0.0"
  port: 8002
  workspace_prefix_key: "[REDACTED]"
  simple_user_map_key: "[REDACTED]"
  jwt:
    secret_signing_key: "[REDACTED]"
    expiration_minutes: 180

database:
  host: ""
  port: 5432
  username: "[REDACTED]"
  password: "[REDACTED]"
  # ... 更多配置
```

---

## 更新 YAML 配置

用新内容替换整个 YAML 配置文件。覆盖前会创建现有配置的备份。

```bash
curl -X PUT http://localhost:8002/osm/api/settings/yaml \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: text/yaml" \
  --data-binary @new-config.yaml
```

**请求体：** 原始 YAML 配置内容

**响应：**
```json
{
  "message": "Configuration updated successfully",
  "path": "/home/user/osmedeus-base/osm-settings.yaml",
  "backup": "/home/user/osmedeus-base/osm-settings.yaml.backup"
}
```

**错误响应（无效 YAML）：**
```json
{
  "error": true,
  "message": "Invalid YAML configuration: yaml: unmarshal errors: ..."
}
```

---

## 重新加载配置

触发配置文件的热重载，无需重启服务器。

```bash
curl -X POST http://localhost:8002/osm/api/settings/reload \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Configuration reloaded successfully",
  "path": "/home/user/osmedeus-base/osm-settings.yaml"
}
```

**错误响应：**
```json
{
  "error": true,
  "message": "Failed to reload configuration: ..."
}
```

---

## 获取配置状态

获取可热重载配置的当前状态，包括上次重载时间和任何错误。

```bash
curl http://localhost:8002/osm/api/settings/status \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "path": "/home/user/osmedeus-base/osm-settings.yaml",
  "last_reload": "2025-01-15T10:30:00Z",
  "is_watching": true,
  "error": ""
}
```

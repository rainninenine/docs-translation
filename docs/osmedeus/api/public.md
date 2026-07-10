---
title: "公共端点"
description: "服务器信息、健康检查和 Swagger 文档"
---

# 公共端点

这些端点不需要身份验证。

## 服务器信息

获取服务器版本和信息。

```bash
curl http://localhost:8002/server-info
```

**响应：**
```json
{
  "message": "Oh dear me, how delightful to notice you're taking a look at this! I'm ever so pleased to let you know that osmedeus is ticking along quite nicely, thank you.",
  "version": "v5.0.0",
  "repo": "https://github.com/j3ssie/osmedeus",
  "author": "j3ssie",
  "docs": "https://docs.osmedeus.org"
}
```

---

## 健康检查

检查服务器是否正在运行。

```bash
curl http://localhost:8002/health
```

**响应：**
```json
{
  "status": "ok"
}
```

---

## 就绪检查

检查服务器是否已准备好接受请求。

```bash
curl http://localhost:8002/health/ready
```

**响应：**
```json
{
  "status": "ready"
}
```

---

## Swagger 文档

访问交互式 Swagger UI 文档。

```bash
# 在浏览器中打开
open http://localhost:8002/swagger/index.html
```

---

## Web UI

Web UI 在根路径提供服务。默认使用嵌入的 UI 文件，也可以选择从外部路径提供服务。

```bash
# 在浏览器中访问 Web UI
open http://localhost:8002/
```

**UI 服务优先级：**
1. 如果配置了 `ui_path` 且路径存在，则从该目录提供服务
2. 否则，从 `public/ui/` 提供嵌入的 UI 文件

---

## 项目空间文件

扫描输出文件可以通过项目空间路径直接访问。此端点仅在服务器设置中配置了 `workspace_prefix` 时可用。

```bash
# 访问运行输出（无需身份验证）
curl http://localhost:8002/ws/{workspace_prefix}/example.com/subdomain/final.txt
```

项目空间路径从已配置的项目空间目录提供文件服务，并启用目录列表。

---
title: "Osmedeus API 文档"
description: "用于管理安全自动化工作流的 RESTful API 参考"
---

# Osmedeus API 文档

## 概述

Osmedeus API 提供了用于管理安全自动化工作流、运行实例和分布式任务执行的 RESTful 接口。

**基础 URL：** `http://localhost:8002`

**默认端口：** `8002`

## 身份验证

大多数 API 端点需要身份验证。支持两种方法：

1. **JWT 令牌**：通过登录端点获取令牌，然后在请求中使用 `Authorization: Bearer ***` 头。

2. **API 密钥**：通过 `x-osm-api-key` 头使用静态 API 密钥。在 `~/osmedeus-base/osm-settings.yaml` 的 `server.auth_api_key` 下配置。

详见 [身份验证](authentication.mdx)。

## API 参考

| 分类 | 描述 |
|----------|-------------|
| [公共端点](public.mdx) | 服务器信息、健康检查、Swagger 文档 |
| [身份验证](authentication.mdx) | 登录、登出和 JWT 令牌管理 |
| [工作流](workflows.mdx) | 列出、查看和刷新工作流 |
| [运行实例](runs.mdx) | 创建和管理工作流执行 |
| [文件上传](uploads.mdx) | 上传目标文件和工作流 |
| [状态快照](snapshots.mdx) | 导出和导入项目空间快照 |
| [项目空间](workspaces.mdx) | 列出和管理项目空间 |
| [产物](artifacts.mdx) | 列出和下载输出产物 |
| [资产](assets.mdx) | 查看已发现的资产 |
| [漏洞](vulnerabilities.mdx) | 查看和管理漏洞 |
| [事件日志](event-logs.mdx) | 查看执行事件日志 |
| [步骤结果](steps.mdx) | 查询步骤执行结果 |
| [函数](functions.mdx) | 执行和列出工具函数 |
| [系统统计](system.mdx) | 获取聚合系统统计信息 |
| [设置](settings.mdx) | 管理服务器配置 |
| [数据库](database.mdx) | 数据库管理和清理 |
| [安装](install.mdx) | 安装二进制文件和工作流 |
| [调度](schedules.mdx) | 管理定时工作流 |
| [事件接收器](event-receiver.mdx) | 事件触发的工作流 |
| [分布式模式](distributed.mdx) | 工作节点和任务管理 |
| [LLM API](llm.mdx) | 大语言模型 API |
| [参考](reference.mdx) | 错误码、分页、Cron 表达式、步骤类型 |

## 快速开始

```bash
# 获取服务器信息（无需身份验证）
curl http://localhost:8002/server-info

# 登录并获取令牌
export TOKEN=$(curl -s -X POST http://localhost:8002/osm/api/login \
  -H "Content-Type: application/json" \
  -d '{"username": "osmedeus", "password": "admin"}' | jq -r '.token')

# 列出工作流
curl http://localhost:8002/osm/api/workflows \
  -H "Authorization: Bearer ***"

# 启动扫描
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{"flow": "subdomain-enum", "target": "example.com"}'
```

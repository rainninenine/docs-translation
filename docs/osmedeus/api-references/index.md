> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# Osmedeus API 文档

> 用于管理安全自动化工作流的 RESTful API 参考

# Osmedeus API 文档

## 概述

Osmedeus API 提供用于管理安全自动化工作流、运行和分布式任务执行的 RESTful 接口。

**基础 URL：** `http://localhost:8002`

**默认端口：** `8002`

## 身份验证

大多数 API 端点需要身份验证。支持两种方法：

1. **JWT 令牌**：通过登录端点获取令牌，然后在请求中使用 `Authorization: Bearer <token>` 标头包含它。

2. **API 密钥**：通过 `x-osm-api-key` 标头使用静态 API 密钥。在 `~/osmedeus-base/osm-settings.yaml` 的 `server.auth_api_key` 下配置。

详见 [身份验证](/api-references/authentication)。

## API 参考

| 类别                                           | 描述                                           |
| ---------------------------------------------- | ---------------------------------------------- |
| [公共端点](/api-references/public)             | 服务器信息、健康检查、Swagger 文档              |
| [身份验证](/api-references/authentication)     | 登录、注销和 JWT 令牌管理                      |
| [工作流](/api-references/workflows)            | 列出、查看和刷新工作流                         |
| [运行](/api-references/runs)                   | 创建和管理工作流执行                           |
| [文件上传](/api-references/uploads)            | 上传目标文件和工作流                           |
| [快照](/api-references/snapshots)              | 导出和导入项目空间快照                         |
| [项目空间](/api-references/workspaces)         | 列出和管理项目空间                             |
| [产物](/api-references/artifacts)              | 列出和下载输出产物                             |
| [资产](/api-references/assets)                 | 查看发现的资产                                 |
| [漏洞](/api-references/vulnerabilities)        | 查看和管理漏洞                                 |
| [事件日志](/api-references/event-logs)         | 查看执行事件日志                               |
| [步骤结果](/api-references/steps)              | 查询步骤执行结果                               |
| [函数](/api-references/functions)              | 执行和列出实用函数                             |
| [系统统计](/api-references/system)             | 获取聚合系统统计信息                           |
| [设置](/api-references/settings)               | 管理服务器配置                                 |
| [数据库](/api-references/database)             | 数据库管理和清理                               |
| [安装](/api-references/install)                | 安装二进制文件和工作流                         |
| [调度](/api-references/schedules)              | 管理调度工作流                                 |
| [事件接收器](/api-references/event-receiver)   | 事件触发的工作流                               |
| [分布式模式](/api-references/distributed)      | 工作节点和任务管理                             |
| [LLM API](/api-references/llm)                 | 大语言模型 API                                 |
| [参考](/api-references/reference)              | 错误码、分页、cron 表达式、步骤类型            |

## 快速开始

```bash theme={null}
# 获取服务器信息（无需身份验证）
curl http://localhost:8002/server-info

# 登录并获取令牌
export TOKEN=$(curl -s -X POST http://localhost:8002/osm/api/login \
  -H "Content-Type: application/json" \
  -d '{"username": "osmedeus", "password": "admin"}' | jq -r '.token')

# 列出工作流
curl http://localhost:8002/osm/api/workflows \
  -H "Authorization: Bearer $TOKEN"

# 启动扫描
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"flow": "subdomain-enum", "target": "example.com"}'
```
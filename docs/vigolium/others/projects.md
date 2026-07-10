文档API参考其他项目与多租户隔离复制页面使用项目按任务隔离扫描数据。每次扫描、漏洞发现、HTTP记录、范围规则和OAST交互都标记有项目UUID，因此多个任务可以共享一个数据库而不会跨边界泄露。复制页面​Overview
Vigolium 支持基于项目的数据隔离。每条扫描记录、漏洞发现、范围规则、源代码仓库和OAST交互都标记有 `project_uuid`，因此多个任务可以共享同一个数据库而不会跨边界泄露数据。

| 概念 | 含义 |
|:-----|:-----|
| 项目 | 所有扫描数据的命名容器。包含UUID、名称、描述、可选的访问控制列表以及可选的每个项目配置覆盖。 |
| 默认项目 | 在 `vigolium init` 期间创建的内置项目（`00000000-0000-0000-0000-000000000001`）。除非另行指定，否则所有数据都属于该项目。 |
| 项目配置 | 位于 `~/.vigolium/projects/<uuid>/config.yaml` 的可选YAML覆盖层，合并到全局配置之上。 |
| 访问控制 | `allowed_domains`（电子邮件域名模式，如 `@acme.com`）和 `allowed_emails`（精确地址）用于限制谁可以访问项目。参见访问控制。 |

​Managing Projects from the CLI
`vigolium project` 子命令用于创建和管理项目。

​Create a project
```bash
vigolium project create my-engagement
# 已创建项目 my-engagement
#   UUID: a1b2c3d4-...
#   配置: ~/.vigolium/projects/a1b2c3d4-.../config.yaml

# 带描述创建
vigolium project create client-app --description "Q1 2026 pentest for client-app"
```

​List projects
```bash
vigolium project list
# 或
vigolium project ls
```
活动项目以 `*` 标记。

​Set the active project
```bash
eval $(vigolium project use a1b2c3d4-...)
# 活动项目: my-engagement (a1b2c3d4-...)
```
这将导出 `VIGOLIUM_PROJECT_UUID` 环境变量到当前 shell，因此该 shell 中的后续所有命令都使用此项目。

​View the project config path
```bash
vigolium project config
# 或针对特定项目
vigolium project config a1b2c3d4-...
```

​Manage project access
```bash
# 添加允许的域名和电子邮件（自动按格式检测）
vigolium project allow a1b2c3d4-... @acme.com @partner.io user@example.com
# ✓ 已向项目 my-engagement 添加 2 个域名和 1 个电子邮件

# @ 前缀的值归入域名，其余归入电子邮件
vigolium project allow a1b2c3d4-... @newdomain.io another@example.com

# 从两个列表中移除条目
vigolium project remove-access a1b2c3d4-... @partner.io user@example.com
```

​Delete a project
```bash
# 删除项目及其所有关联数据（提示确认）
vigolium project delete a1b2c3d4-...
# 'rm' 和 'remove' 是别名
vigolium project rm a1b2c3d4-...

# 跳过确认提示
vigolium project delete a1b2c3d4-... -F

# 删除项目数据但保留其配置目录在磁盘上
vigolium project delete a1b2c3d4-... --keep-config -F
```
删除操作会移除所有归属于该项目的扫描数据（扫描、HTTP记录、漏洞发现、范围、OAST交互）。默认情况下，项目的配置目录（`~/.vigolium/projects/<uuid>/`）也会被移除；`--keep-config` 则保留该目录。

设置 `VIGOLIUM_PROJECT_READONLY=true` 可禁用 CLI 中所有修改项目的命令（create、allow、remove-access）。只读命令（list、use、config）仍然有效。在生产环境或共享环境中，项目应仅通过 REST API 管理时，此设置非常有用。

​Scoping Operations to a Project
有几种机制可以选择活动项目，按优先级从高到低列出：

| 方法 | 示例 |
|:-----|:-----|
| `--project-uuid` 标志 | `vigolium scan -t https://example.com --project-uuid a1b2c3d4-...` |
| `--project-name` 标志 | `vigolium scan -t https://example.com --project-name my-engagement` |
| `VIGOLIUM_PROJECT_UUID` 环境变量 | `export VIGOLIUM_PROJECT_UUID=a1b2c3d4-...` |
| `VIGOLIUM_PROJECT` 环境变量（旧版） | `export VIGOLIUM_PROJECT=a1b2c3d4-...` |
| 默认项目 | 当未设置标志或环境变量时使用 |

`--project-uuid` 和 `--project-name` 互斥（`--project-name` 必须精确匹配一个项目）。

​CLI examples
```bash
# 在项目内扫描（按UUID或名称）
vigolium scan -t https://example.com --project-uuid a1b2c3d4-...
vigolium scan -t https://example.com --project-name my-engagement

# 导入到项目
vigolium ingest --input urls.txt --project-uuid a1b2c3d4-...

# 查询和导出项目数据
vigolium finding list --project-name my-engagement
vigolium db export --project-uuid a1b2c3d4-... -o findings.jsonl
```

​Server API
使用 REST API 时，设置 `X-Project-UUID` 标头以将所有操作限定到某个项目：
```bash
curl -X POST http://localhost:9002/api/ingest-http \
  -H "Authorization: Bearer my-secret-key" \
  -H "X-Project-UUID: a1b2c3d4-..." \
  -H "Content-Type: application/json" \
  -d '{"input_mode": "url", "content": "https://example.com"}'
```
如果省略该标头，则使用默认项目。所有查询（漏洞发现、HTTP记录、统计信息、扫描）都返回限定到该项目的数据。

​Config Merge Strategy
配置按层次解析（后一层覆盖前一层）：
```
内置默认值
  → ~/.vigolium/vigolium-configs.yaml          (全局配置)
    → ~/.vigolium/projects/<uuid>/config.yaml  (项目配置覆盖)
      → --scanning-profile 标志                (扫描配置文件)
        → CLI 标志                            (最高优先级)
```
项目配置文件使用与扫描配置文件相同的部分YAML格式——仅覆盖您指定的字段：
```yaml
# ~/.vigolium/projects/a1b2c3d4-.../config.yaml
scope:
  hosts:
    - "*.example.com"

scanning_pace:
  concurrency: 30
  rate_limit: 50

dynamic-assessment:
  extensions:
    enabled: true
    variables:
      auth_token: "Bearer project-specific-token"
```
有关完整的配置部分，请参见设置。

​Access Control
项目可以通过 `allowed_domains` 和 `allowed_emails` 字段按电子邮件域名或精确电子邮件地址限制访问。

​How it works
当请求同时包含 `X-Project-UUID` 和 `X-User-Email` 标头时，服务器按以下顺序检查访问权限：

1. 如果 `allowed_emails` 非空 → 用户的电子邮件必须精确匹配（不区分大小写）。
2. 否则，如果 `allowed_domains` 非空 → 用户的电子邮件域名（例如 `@acme.com`）必须匹配。
3. 如果两个列表都为空 → 项目对任何人开放。
4. 如果未发送 `X-User-Email` → 完全跳过检查。

被拒绝的请求将收到 `403 Forbidden` 响应。

​Managing via API
```bash
# 创建时设置访问列表
curl -X POST http://localhost:9002/api/projects \
  -H "Content-Type: application/json" \
  -d '{"name":"restricted","allowed_domains":["@acme.com"],"allowed_emails":["user@example.com"]}'

# 更新访问列表
curl -X PUT http://localhost:9002/api/projects/a1b2c3d4-... \
  -H "Content-Type: application/json" \
  -d '{"allowed_domains":["@acme.com","@partner.io"]}'

# 清除限制（项目变为开放）
curl -X PUT http://localhost:9002/api/projects/a1b2c3d4-... \
  -H "Content-Type: application/json" \
  -d '{"allowed_domains":[],"allowed_emails":[]}'

# 域名到项目的映射（用于前端中间件）
curl http://localhost:9002/api/projects/domain-map
```
有关完整端点参考，请参见项目API。

​Database Isolation
所有主要数据表都包含 `project_uuid` 列，并在 CLI、服务器 API 和内部管道中按活动项目进行过滤：
```
scans · http_records · findings · scopes · source_repos · oast_interactions · scan_logs
```
现有数据库会自动迁移——`project_uuid` 列会被添加，默认值为默认项目的 UUID，因此项目前的数据在默认项目下仍然可访问。上一页开源审计展示来自流行开源项目的真实漏洞扫描报告，由 Vigolium 的 agentic 扫描引擎驱动下一页⌘IwebsitetwittergithubdiscordxlinkedinPowered by本文档基于 Mintlify 构建和托管，一个开发者文档平台本页内容OverviewManaging Projects from the CLICreate a projectList projectsSet the active projectView the project config pathManage project accessDelete a projectScoping Operations to a ProjectCLI examplesServer APIConfig Merge StrategyAccess ControlHow it worksManaging via APIDatabase Isolation
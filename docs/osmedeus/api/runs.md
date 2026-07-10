---
title: "运行实例（扫描）"
description: "创建和管理工作流执行"
---

# 运行实例（扫描）

## 创建新扫描

针对目标执行工作流。

**使用 flow 工作流的基本扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "target": "example.com"
  }'
```

**使用 module 工作流的基本扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "module": "port-scan",
    "target": "example.com"
  }'
```

**带自定义参数的扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "target": "example.com",
    "params": {
      "threads": "50",
      "timeout": "30"
    }
  }'
```

**带优先级和超时的扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "target": "example.com",
    "priority": "high",
    "timeout": 60
  }'
```

**使用 Docker 运行器的扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "target": "example.com",
    "runner_type": "docker",
    "docker_image": "osmedeus/osmedeus:latest"
  }'
```

**使用 SSH 运行器的扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "target": "example.com",
    "runner_type": "ssh",
    "ssh_host": "worker1.example.com"
  }'
```

**请求体参数：**

| 参数 | 类型 | 必填 | 默认值 | 描述 |
|-----------|------|----------|---------|-------------|
| `flow` | string | 否* | - | 要执行的 Flow 工作流名称 |
| `module` | string | 否* | - | 要执行的 Module 工作流名称 |
| `target` | string | 否** | - | 要扫描的单个目标 |
| `targets` | array | 否** | - | 要扫描的多个目标 |
| `target_file` | string | 否** | - | 包含目标的文件路径 |
| `params` | object | 否 | `{}` | 自定义工作流参数 |
| `priority` | string | 否 | `normal` | 优先级：`low`、`normal`、`high`、`critical` |
| `timeout` | int | 否 | - | 运行超时时间（分钟） |
| `concurrency` | int | 否 | 1 | 并发目标数 |
| `runner_type` | string | 否 | `host` | 执行环境：`host`、`docker`、`ssh` |
| `docker_image` | string | 否 | - | Docker 运行器的 Docker 镜像 |
| `ssh_host` | string | 否 | - | SSH 运行器的 SSH 主机 |
| `workspace` | string | 否 | auto | 自定义项目空间名称 |
| `run_mode` | string | 否 | `local` | 执行模式：`local`、`distributed`、`cloud` |
| `cloud_provider` | string | 否 | config default | 云供应商：`aws`、`gcp`、`digitalocean`、`linode`、`azure`、`hetzner` |
| `cloud_instances` | int | 否 | 1 | 要预配的云实例数 |
| `cloud_instance_type` | string | 否 | provider default | 实例大小覆盖（例如 `t3.medium`、`s-2vcpu-4gb`） |
| `cloud_region` | string | 否 | provider default | 区域覆盖 |
| `cloud_auto_destroy` | bool | 否 | `false` | 扫描完成后销毁云基础设施 |
| `cloud_reuse_infra` | string | 否 | - | 要复用的现有基础设施 ID，而非重新预配 |
| `cloud_use_spot` | bool | 否 | `false` | 使用竞价/抢占式实例以节省成本 |

\* `flow` 或 `module` 中必须指定一个。
\** `target`、`targets` 或 `target_file` 中必须指定一个。

**响应：**
```json
{
  "message": "Run started",
  "workflow": "subdomain-enum",
  "kind": "flow",
  "target": "example.com",
  "target_count": 1,
  "priority": "normal",
  "run_mode": "local",
  "job_id": "a1b2c3d4",
  "run_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "poll_url": "/osm/api/jobs/a1b2c3d4",
  "runner_type": "docker",
  "timeout": 60
}
```

---

## 多目标扫描

通过并发控制扫描多个目标：

```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "targets": ["example.com", "test.com", "demo.com"],
    "concurrency": 3
  }'
```

**响应：**
```json
{
  "message": "Run started",
  "workflow": "subdomain-enum",
  "kind": "flow",
  "target_count": 3,
  "targets": ["example.com", "test.com", "demo.com"],
  "concurrency": 3,
  "priority": "normal",
  "job_id": "b2c3d4e5",
  "status": "queued",
  "poll_url": "/osm/api/jobs/b2c3d4e5"
}
```

---

## 从上传的目标文件扫描

使用已上传的目标文件（来自 `/osm/api/upload-file`）进行扫描：

```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "module": "port-scan",
    "target_file": "/home/user/osmedeus-base/data/uploads/targets.txt",
    "concurrency": 5
  }'
```

这类似于 CLI 的 `-T` 标志：`osmedeus run -m port-scan -T targets.txt`

---

## 分布式模式

向分布式工作节点池提交扫描。需要服务器以 `--master` 标志启动。

**单目标分布式扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "target": "example.com",
    "run_mode": "distributed"
  }'
```

**跨工作节点多目标扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "targets": ["example.com", "test.com", "demo.com"],
    "run_mode": "distributed",
    "priority": "high"
  }'
```

每个目标作为单独的任务提交到分布式工作节点队列。工作节点获取任务并独立执行。

**响应：**
```json
{
  "message": "Run started",
  "workflow": "subdomain-enum",
  "kind": "flow",
  "target_count": 3,
  "targets": ["example.com", "test.com", "demo.com"],
  "priority": "high",
  "run_mode": "distributed",
  "job_id": "c3d4e5f6",
  "status": "queued",
  "poll_url": "/osm/api/jobs/c3d4e5f6"
}
```

**服务器未处于主节点模式时的错误：**
```json
{
  "error": true,
  "message": "Distributed mode requires the server to be started with --master flag"
}
```

---

## 云模式

预配云基础设施并在远程实例上执行扫描。需要在配置中启用云功能（`osm-settings.yaml` 中的 `cloud.enabled: true`）。

**基本云扫描（使用配置中的默认供应商）：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "target": "example.com",
    "run_mode": "cloud"
  }'
```

**多实例云扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "targets": ["example.com", "test.com", "demo.com"],
    "run_mode": "cloud",
    "cloud_provider": "digitalocean",
    "cloud_instances": 3,
    "cloud_auto_destroy": true
  }'
```

目标以轮询方式分配到已预配的实例。当 `cloud_auto_destroy` 为 `true` 时，所有扫描完成后基础设施将被拆除。

**带实例自定义的云扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "general",
    "target": "example.com",
    "run_mode": "cloud",
    "cloud_provider": "aws",
    "cloud_instances": 2,
    "cloud_instance_type": "t3.large",
    "cloud_region": "ap-southeast-1",
    "cloud_use_spot": true,
    "cloud_auto_destroy": true,
    "priority": "high"
  }'
```

**复用现有基础设施的云扫描：**
```bash
curl -X POST http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "flow": "subdomain-enum",
    "target": "example.com",
    "run_mode": "cloud",
    "cloud_reuse_infra": "a1b2c3d4"
  }'
```

通过传递 `cloud_reuse_infra` ID（来自之前的 `POST /cloud/instances` 或云运行响应）跳过预配，在现有基础设施上运行。

**响应：**
```json
{
  "message": "Run started",
  "workflow": "subdomain-enum",
  "kind": "flow",
  "target": "example.com",
  "target_count": 1,
  "priority": "high",
  "run_mode": "cloud",
  "job_id": "d4e5f6a7",
  "run_uuid": "770e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "poll_url": "/osm/api/jobs/d4e5f6a7",
  "infra_id": "d4e5f6a7",
  "infra_status_url": "/osm/api/cloud/instances/d4e5f6a7/status",
  "cloud_provider": "aws",
  "cloud_instances": 2
}
```

**云功能未启用时的错误：**
```json
{
  "error": true,
  "message": "Cloud mode requires cloud features to be enabled in configuration"
}
```

**供应商凭据无效时的错误：**
```json
{
  "error": true,
  "message": "Cloud provider validation failed: invalid API token"
}
```

---

## 列出运行实例

获取所有运行实例的分页列表。

```bash
curl http://localhost:8002/osm/api/runs \
  -H "Authorization: Bearer ***"
```

**查询参数：**

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------|-------------|
| `status` | string | - | 按状态过滤：`pending`、`running`、`completed`、`failed` |
| `workflow_name` | string | - | 按工作流名称过滤 |
| `target` | string | - | 按目标过滤 |
| `offset` | int | 0 | 分页偏移量 |
| `limit` | int | 20 | 最大返回记录数 |

**响应：**
```json
{
  "data": [
    {
      "id": 1,
      "run_uuid": "550e8400-e29b-41d4-a716-446655440000",
      "workflow_name": "subdomain-enum",
      "workflow_kind": "flow",
      "target": "example.com",
      "params": {"threads": "50"},
      "status": "running",
      "workspace": "example.com",
      "started_at": "2025-01-15T10:00:00Z",
      "completed_at": null,
      "total_steps": 10,
      "completed_steps": 3,
      "current_pid": 12345,
      "trigger_type": "manual",
      "run_group_id": "a1b2c3d4",
      "created_at": "2025-01-15T10:00:00Z",
      "updated_at": "2025-01-15T10:03:00Z"
    }
  ],
  "pagination": {
    "total": 50,
    "offset": 0,
    "limit": 20
  }
}
```

**注意：** `current_pid` 字段显示当前正在运行的命令的进程 ID。可用于识别和取消正在运行的进程。当运行完成时，此字段将被清除（设置为 0 或省略）。

---

## 获取运行详情

通过 ID 获取特定运行的详细信息。

```bash
curl http://localhost:8002/osm/api/runs/run-abc123 \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": {
    "id": 1,
    "run_uuid": "550e8400-e29b-41d4-a716-446655440000",
    "workflow_name": "subdomain-enum",
    "workflow_kind": "flow",
    "target": "example.com",
    "params": {"threads": "50"},
    "status": "completed",
    "workspace": "example.com",
    "started_at": "2025-01-15T10:00:00Z",
    "completed_at": "2025-01-15T10:30:00Z",
    "error_message": "",
    "schedule_id": "",
    "trigger_type": "manual",
    "trigger_name": "",
    "run_group_id": "a1b2c3d4",
    "total_steps": 10,
    "completed_steps": 10,
    "created_at": "2025-01-15T10:00:00Z",
    "updated_at": "2025-01-15T10:30:00Z"
  }
}
```

**注意：** 您可以使用数字 `id` 或 `run_uuid` 来获取运行详情。

---

## 取消运行

取消正在运行的工作流执行。这将终止与该运行关联的所有正在运行的进程。

```bash
# 按 run_uuid 取消
curl -X DELETE http://localhost:8002/osm/api/runs/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer ***"

# 或按数字 id 取消
curl -X DELETE http://localhost:8002/osm/api/runs/1 \
  -H "Authorization: Bearer ***"
```

**响应（进程已成功终止）：**
```json
{
  "message": "Run cancelled successfully",
  "id": 1,
  "run_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "killed_pids": [12345, 12346],
  "processes_terminated": 2,
  "kill_method": "registry"
}
```

**响应（使用数据库 PID 回退）：**
```json
{
  "message": "Run cancelled successfully",
  "id": 1,
  "run_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "killed_pids": [12345],
  "processes_terminated": 1,
  "kill_method": "database_pid"
}
```

**响应（未找到活动进程）：**
```json
{
  "message": "Run cancelled successfully",
  "id": 1,
  "run_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "note": "No active processes found to terminate; database status updated"
}
```

**终止方法：**
- `registry` - 进程在内存中跟踪并通过运行注册表终止（API 发起的运行）
- `database_pid` - 使用存储在数据库中的 PID 终止进程（回退方法）

---

## 获取运行步骤

获取特定运行的所有步骤结果。

```bash
# 使用 run_uuid
curl http://localhost:8002/osm/api/runs/550e8400-e29b-41d4-a716-446655440000/steps \
  -H "Authorization: Bearer ***"

# 或使用数字 id
curl http://localhost:8002/osm/api/runs/1/steps \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "run_id": 1,
      "step_name": "run-subfinder",
      "step_type": "bash",
      "status": "completed",
      "command": "subfinder -d example.com -o subdomains.txt",
      "output": "Found 150 subdomains",
      "error_message": "",
      "exports": {"subdomains_file": "subdomains.txt"},
      "duration_ms": 45000,
      "log_file": "/workspaces/example.com/logs/run-subfinder.log",
      "started_at": "2025-01-15T10:01:00Z",
      "completed_at": "2025-01-15T10:01:45Z",
      "created_at": "2025-01-15T10:01:00Z"
    },
    {
      "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "run_id": 1,
      "step_name": "run-httpx",
      "step_type": "bash",
      "status": "completed",
      "command": "httpx -l subdomains.txt -o alive.txt",
      "output": "Probed 150 hosts, 89 alive",
      "error_message": "",
      "exports": {"alive_file": "alive.txt"},
      "duration_ms": 120000,
      "log_file": "/workspaces/example.com/logs/run-httpx.log",
      "started_at": "2025-01-15T10:01:45Z",
      "completed_at": "2025-01-15T10:03:45Z",
      "created_at": "2025-01-15T10:01:45Z"
    }
  ]
}
```

---

## 获取运行产物

获取特定运行的所有输出产物。

```bash
# 使用 run_uuid
curl http://localhost:8002/osm/api/runs/550e8400-e29b-41d4-a716-446655440000/artifacts \
  -H "Authorization: Bearer ***"

# 或使用数字 id
curl http://localhost:8002/osm/api/runs/1/artifacts \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "data": [
    {
      "id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
      "run_id": 1,
      "workspace": "example.com",
      "name": "subdomains.txt",
      "artifact_path": "/workspaces/example.com/subdomains.txt",
      "artifact_type": "output",
      "content_type": "txt",
      "size_bytes": 4523,
      "line_count": 150,
      "description": "Discovered subdomains",
      "created_at": "2025-01-15T10:01:45Z"
    },
    {
      "id": "d4e5f6a7-b8c9-0123-def0-234567890123",
      "run_id": 1,
      "workspace": "example.com",
      "name": "alive.txt",
      "artifact_path": "/workspaces/example.com/alive.txt",
      "artifact_type": "output",
      "content_type": "txt",
      "size_bytes": 2890,
      "line_count": 89,
      "description": "Alive HTTP endpoints",
      "created_at": "2025-01-15T10:03:45Z"
    },
    {
      "id": "e5f6a7b8-c9d0-1234-ef01-345678901234",
      "run_id": 1,
      "workspace": "example.com",
      "name": "nuclei-results.json",
      "artifact_path": "/workspaces/example.com/nuclei-results.json",
      "artifact_type": "output",
      "content_type": "json",
      "size_bytes": 15234,
      "line_count": 45,
      "description": "Nuclei vulnerability scan results",
      "created_at": "2025-01-15T10:15:00Z"
    }
  ]
}
```

**产物类型：**
- `report` - 从工作流的报告部分生成的报告
- `state_file` - 状态文件，如 run-state.json、run-execution.log
- `output` - 步骤的通用输出文件
- `screenshot` - 扫描期间捕获的截图

**内容类型：**
- `json`、`jsonl`、`yaml`、`html`、`md`、`log`、`pdf`、`png`、`txt`、`zip`、`folder`、`unknown`

---

## 复制运行

使用相同配置创建现有运行的副本。

```bash
curl -X POST http://localhost:8002/osm/api/runs/550e8400-e29b-41d4-a716-446655440000/duplicate \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Run duplicated",
  "original_run_id": "550e8400-e29b-41d4-a716-446655440000",
  "new_run_uuid": "660e8400-e29b-41d4-a716-446655440001",
  "status": "pending"
}
```

---

## 启动运行

启动已创建但尚未开始的待处理运行。

```bash
curl -X POST http://localhost:8002/osm/api/runs/550e8400-e29b-41d4-a716-446655440000/start \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "message": "Run started",
  "run_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "status": "running"
}
```

---

## 获取作业状态

获取作业（来自同一请求的一组运行）的状态。这对于跟踪多目标扫描非常有用。

```bash
curl http://localhost:8002/osm/api/jobs/a1b2c3d4 \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "job_id": "a1b2c3d4",
  "total_runs": 3,
  "completed": 2,
  "running": 1,
  "failed": 0,
  "pending": 0,
  "runs": [
    {
      "run_uuid": "550e8400-e29b-41d4-a716-446655440000",
      "target": "example.com",
      "status": "completed"
    },
    {
      "run_uuid": "550e8400-e29b-41d4-a716-446655440001",
      "target": "test.com",
      "status": "completed"
    },
    {
      "run_uuid": "550e8400-e29b-41d4-a716-446655440002",
      "target": "demo.com",
      "status": "running"
    }
  ]
}
```

---

## 优先级级别

可以为运行分配优先级级别，以控制多个运行排队时的执行顺序。

| 优先级 | 描述 |
|----------|-------------|
| `low` | 最低优先级，最后处理 |
| `normal` | 默认优先级（未指定时使用） |
| `high` | 较高优先级，在 normal/low 之前处理 |
| `critical` | 最高优先级，最先处理 |

**注意：** 请求中未指定时，优先级默认为 `normal`。

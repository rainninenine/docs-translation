> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# Docker 安装与设置

> 使用 Docker 进行容器化设置与执行

Osmedeus 提供多种 Docker 部署选项，以适应不同使用场景：

| 选项             | 最佳用途                           | 镜像                       |
| ---------------- | ---------------------------------- | -------------------------- |
| **Toolbox**      | 快速启动、交互式扫描               | `osmedeus-toolbox:latest`  |
| **Production**   | 独立部署、API 服务器               | `osmedeus:latest`          |
| **Distributed**  | 使用 Worker 进行大规模扫描         | Master + Workers           |

## 前提条件

* **Docker** 版本 20.10 或更高
* **Docker Compose** 版本 2.0 或更高（Docker Desktop 已包含）

```bash theme={null}
# 验证安装
docker --version
docker compose version
```

## 使用 Toolbox 快速启动

Toolbox 镜像是开始使用的最快方式。它通过官方安装脚本预装了所有安全工具。

### 使用 Make 目标

```bash theme={null}
# 构建 toolbox 镜像
make docker-toolbox

# 启动容器
make docker-toolbox-run

# 进入交互式 shell
make docker-toolbox-shell
```

### 手动 Docker 命令

```bash theme={null}
# 构建镜像
docker compose -f build/docker/docker-compose.toolbox.yaml build

# 启动容器
docker compose -f build/docker/docker-compose.toolbox.yaml up -d

# 进入容器
docker exec -it osmedeus-toolbox bash

# 直接运行命令
docker exec -it osmedeus-toolbox osmedeus run -m subdomain -t example.com
```

<Note>
  Toolbox 容器使用 Docker 卷持久化 `osmedeus-base` 和 `workspaces-osmedeus` 数据，因此扫描结果在容器重启后仍然保留。
</Note>

## 运行工作流

### 交互模式

进入容器 shell 并交互式运行扫描：

```bash theme={null}
docker exec -it osmedeus-toolbox bash

# 在容器内部
osmedeus run -f extensive -t example.com
osmedeus run -m subdomain -t example.com
osmedeus workflow list
```

### 一次性命令

无需进入容器即可运行扫描：

```bash theme={null}
# 运行模块
docker exec -it osmedeus-toolbox osmedeus run -m subdomain -t example.com

# 运行 Flow
docker exec -it osmedeus-toolbox osmedeus run -f extensive -t example.com

# 列出工作流
docker exec -it osmedeus-toolbox osmedeus workflow list
```

### 使用卷挂载持久化结果

如需在宿主机访问扫描结果，可挂载本地目录：

```bash theme={null}
docker run -it --rm \
  -v $(pwd)/workspaces:/root/workspaces-osmedeus \
  -v $(pwd)/osmedeus-base:/root/osmedeus-base \
  osmedeus-toolbox:latest \
  osmedeus run -m subdomain -t example.com
```

## 运行服务器

### 基本服务器启动

在 toolbox 容器内启动 REST API 服务器：

```bash theme={null}
# 后台启动
docker exec -d osmedeus-toolbox osmedeus serve --port 8002 --host 0.0.0.0

# 或进入容器后运行
docker exec -it osmedeus-toolbox bash
osmedeus serve --port 8002 --host 0.0.0.0
```

### 带身份验证的服务器

API 服务器默认使用 JWT 身份验证。在 `osm-settings.yaml` 中配置凭据：

```yaml theme={null}
server:
  host: "0.0.0.0"
  port: 8002
  simple_user_map_key:
    admin: "your-secure-password"
  jwt:
    secret_signing_key: "your-jwt-secret-min-32-chars"
    expiration_minutes: 1440
```

启用身份验证后启动：

```bash theme={null}
osmedeus serve --port 8002 --host 0.0.0.0
```

开发环境无需身份验证时，使用 `-A` 标志：

```bash theme={null}
osmedeus serve --port 8002 --host 0.0.0.0 -A
```

## 使用 Docker Compose 的分布式模式

对于大规模扫描，使用主节点协调多个 Worker 的分布式架构。

### 架构

```
                    ┌─────────────────────────────────────────┐
                    │              负载均衡器                 │
                    │             (可选)                      │
                    └────────────────┬────────────────────────┘
                                     │
                    ┌────────────────▼────────────────────────┐
                    │           主服务器                      │
                    │  - REST API (端口 8002)                │
                    │  - 任务协调器                          │
                    │  - Web UI                              │
                    └────────────────┬────────────────────────┘
                                     │
                    ┌────────────────▼────────────────────────┐
                    │              Redis                      │
                    │  - 消息队列                            │
                    │  - 任务分发                            │
                    └────────────────┬────────────────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
┌───────▼───────┐          ┌─────────▼───────┐          ┌─────────▼───────┐
│   Worker 1    │          │    Worker 2     │          │    Worker N     │
│  - 执行扫描   │          │  - 执行扫描     │          │  - 执行扫描     │
└───────────────┘          └─────────────────┘          └─────────────────┘
```

### 基本设置

基础 compose 文件（`docker-compose.yml`）包含 Redis、主节点和 Worker，使用 SQLite：

```bash theme={null}
# 启动分布式栈
docker compose -f build/docker/docker-compose.yml up -d

# 扩展 Worker 数量
docker compose -f build/docker/docker-compose.yml up -d --scale worker=5

# 查看日志
docker compose -f build/docker/docker-compose.yml logs -f

# 停止所有服务
docker compose -f build/docker/docker-compose.yml down
```

### 生产环境设置

对于使用 PostgreSQL 的生产部署，请使用 `docker-compose.production.yaml`：

**步骤 1：创建环境文件**

```bash theme={null}
cp build/docker/.env.example build/docker/.env
```

编辑 `.env` 文件，填入安全值：

```bash theme={null}
# PostgreSQL 配置
POSTGRES_USER=osmedeus
POSTGRES_PASSWORD=your_secure_postgres_password
POSTGRES_DB=osmedeus

# Redis（可选）
# REDIS_PASSWORD=your_secure_redis_password

# 服务器
OSM_SERVER_PORT=8002
TZ=UTC

# Worker
WORKER_REPLICAS=2
```

<Note>
  使用以下命令生成安全密码：`openssl rand -base64 24`
</Note>

**步骤 2：配置应用程序设置**

生产环境 compose 文件挂载了 `osm-settings.production.yaml`。关键设置如下：

```yaml theme={null}
# 数据库 - 生产环境使用 PostgreSQL
database:
  db_engine: postgresql
  host: postgres
  port: 5432
  username: osmedeus
  password: "CHANGE_ME_POSTGRES_PASSWORD"  # 必须与 .env 文件一致
  db_name: osmedeus
  ssl_mode: disable

# 服务器
server:
  host: "0.0.0.0"
  port: 8002
  simple_user_map_key:
    admin: "CHANGE_ME_ADMIN_PASSWORD"
  jwt:
    secret_signing_key: "CHANGE_ME_JWT_SECRET_MIN_32_CHARS"
    expiration_minutes: 1440

# Redis（分布式模式必需）
redis:
  host: redis
  port: 6379
  password: ""  # 如果在 .env 中配置了 REDIS_PASSWORD，则设置此项
```

**步骤 3：启动生产环境栈**

```bash theme={null}
# 先启动基础设施
docker compose -f build/docker/docker-compose.production.yaml up -d postgres redis

# 等待健康状态（约 10-15 秒）
docker compose -f build/docker/docker-compose.production.yaml ps

# 启动应用程序
docker compose -f build/docker/docker-compose.production.yaml up -d

# 扩展 Worker 以实现并行扫描
docker compose -f build/docker/docker-compose.production.yaml up -d --scale worker=5
```

### 环境变量

| 变量                | 描述                         | 默认值        |
| ------------------- | ---------------------------- | ------------- |
| `POSTGRES_USER`     | PostgreSQL 用户名            | `osmedeus`    |
| `POSTGRES_PASSWORD` | PostgreSQL 密码              | **必需**      |
| `POSTGRES_DB`       | PostgreSQL 数据库名          | `osmedeus`    |
| `REDIS_PASSWORD`    | Redis 密码（可选）           | -             |
| `OSM_SERVER_PORT`   | 外部 API 端口                | `8002`        |
| `TZ`                | 容器时区                     | `UTC`         |
| `WORKER_REPLICAS`   | Worker 容器数量              | `2`           |

### 扩展 Worker

可根据工作负载动态扩展 Worker：

```bash theme={null}
# 扩展到 10 个 Worker
docker compose -f build/docker/docker-compose.production.yaml up -d --scale worker=10

# 缩减回 2 个 Worker
docker compose -f build/docker/docker-compose.production.yaml up -d --scale worker=2
```

每个 Worker 具有资源限制（可在 compose 文件中配置）：

* CPU：2 核（上限），0.5 核（预留）
* 内存：2GB（上限），512MB（预留）

## 构建自定义镜像

### 生产环境镜像

生产环境 Dockerfile（`build/docker/Dockerfile`）创建最小化镜像：

```bash theme={null}
# 使用 make 构建
make docker-build

# 或直接构建
docker build -t osmedeus:latest \
  -f build/docker/Dockerfile \
  --build-arg VERSION=5.0.0 \
  .
```

### 开发环境镜像

用于热重载的开发环境：

```bash theme={null}
docker build -t osmedeus:dev -f build/docker/Dockerfile.dev .

docker run -it --rm \
  -v $(pwd):/app \
  -p 8002:8002 \
  osmedeus:dev
```

开发环境镜像包含：

* 完整的 Go 工具链
* 用于热重载的 Air
* 用于编辑的 Vim

### Toolbox 镜像

构建包含所有工具的完整功能 Toolbox：

```bash theme={null}
docker build -t osmedeus-toolbox:latest \
  -f build/docker/Dockerfile.toolbox \
  .
```

## 卷管理

Osmedeus 使用两个主要数据目录：

| 卷                | 容器路径                      | 用途                            |
| ----------------- | ----------------------------- | ------------------------------- |
| `osmedeus-data`   | `/root/osmedeus-base`         | 配置、工作流、数据库            |
| `workspaces`      | `/root/workspaces-osmedeus`   | 扫描结果和产物                  |

### 备份卷

```bash theme={null}
# 列出卷
docker volume ls | grep osmedeus

# 备份 workspaces
docker run --rm \
  -v osmedeus-workspaces:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/workspaces-backup.tar.gz -C /data .

# 恢复 workspaces
docker run --rm \
  -v osmedeus-workspaces:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/workspaces-backup.tar.gz -C /data
```

### 删除卷

```bash theme={null}
# 删除所有 osmedeus 卷（警告：将删除所有数据）
docker compose -f build/docker/docker-compose.production.yaml down -v
```

## 故障排除

### 容器无法启动

```bash theme={null}
# 检查容器日志
docker logs osmedeus-toolbox

# 检查 compose 日志
docker compose -f build/docker/docker-compose.yml logs -f master
```

### 权限问题

如果遇到挂载卷的权限问题：

```bash theme={null}
# 以 root 用户运行容器（默认）
docker exec -u root -it osmedeus-toolbox bash

# 或修复宿主机目录权限
sudo chown -R $(id -u):$(id -g) ./workspaces
```

### 数据库连接错误

对于 PostgreSQL 连接问题：

```bash theme={null}
# 检查 PostgreSQL 是否健康
docker compose -f build/docker/docker-compose.production.yaml ps postgres

# 从主节点测试连接
docker exec osmedeus-server psql -h postgres -U osmedeus -d osmedeus -c "SELECT 1"
```

### Redis 连接错误

```bash theme={null}
# 检查 Redis 是否健康
docker exec osmedeus-redis redis-cli ping

# 使用密码测试
docker exec osmedeus-redis redis-cli -a your_password ping
```

### 健康检查失败

```bash theme={null}
# 检查主节点健康端点
curl http://localhost:8002/health

# 查看健康检查日志
docker inspect --format='{{json .State.Health}}' osmedeus-server | jq
```

### 内存不足

如果 Worker 内存不足，可在 compose 文件中调整资源限制：

```yaml theme={null}
worker:
  deploy:
    resources:
      limits:
        memory: 4G
      reservations:
        memory: 1G
```
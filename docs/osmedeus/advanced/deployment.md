# Deployment

> 构建、部署和运行 Osmedeus 于各种环境

本指南涵盖在各种环境中构建、部署和运行 Osmedeus。

## 先决条件

* Go 1.21+（用于本地构建）
* Docker 20.10+（用于容器化部署）
* Docker Compose 2.0+（用于分布式模式）

## 快速开始

```bash
# 本地构建并运行
make build
./build/bin/osmedeus serve

# Docker 单容器
docker build -t osmedeus:latest -f build/docker/Dockerfile .
docker run -p 8001:8001 osmedeus:latest

# 使用 Docker Compose 的分布式模式
docker-compose -f build/docker/docker-compose.yml up -d
```

## 构建

### 本地构建

```bash
# 为当前平台构建
make build

# 跨平台构建
make build-all       # 所有平台
make build-linux     # Linux amd64
make build-darwin    # macOS amd64 + arm64
make build-windows   # Windows amd64

# 输出位置
./build/bin/osmedeus
```

### Docker 构建

```bash
docker build -t osmedeus:latest -f build/docker/Dockerfile .

# 开发镜像（带热重载）
docker build -t osmedeus:dev -f build/docker/Dockerfile.dev .

# 自定义版本
docker build --build-arg VERSION=5.1.0 -t osmedeus:5.1.0 -f build/docker/Dockerfile .
```

## 部署模式

### 单主机

#### 直接二进制

```bash
# 运行服务器
./build/bin/osmedeus serve --port 8001

# 禁用身份验证运行（仅开发环境）
./build/bin/osmedeus serve -A

# 运行扫描
./build/bin/osmedeus scan -f general -t example.com
```

#### Docker 容器

```bash
# 基础服务器
docker run -d \
  --name osmedeus \
  -p 8001:8001 \
  -v osmedeus-data:/root/osmedeus-base \
  -v workspaces:/root/workspaces-osmedeus \
  osmedeus:latest

# 使用自定义工作流
docker run -d \
  --name osmedeus \
  -p 8001:8001 \
  -v /path/to/workflows:/root/osmedeus-base/workflows \
  -v /path/to/workspaces:/root/workspaces-osmedeus \
  osmedeus:latest
```

### 分布式模式（主/从）

分布式模式允许通过使用 Redis 作为消息队列，在多个工作节点上扩展扫描工作负载。

#### 体系结构

```
                    ┌─────────────┐
                    │   客户端    │
                    └──────┬──────┘
                           │ REST API
                    ┌──────▼──────┐
                    │    主节点   │
                    │  (服务器)   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    Redis    │
                    │   (队列)    │
                    └──────┬──────┘
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼────┐ ┌─────▼────┐ ┌─────▼────┐
        │ 工作节点1│ │ 工作节点2│ │ 工作节点N│
        └──────────┘ └──────────┘ └──────────┘
```

#### Docker Compose 设置

```bash
# 启动 2 个工作节点（默认）
docker-compose -f build/docker/docker-compose.yml up -d

# 扩展到 5 个工作节点
docker-compose -f build/docker/docker-compose.yml up -d --scale worker=5

# 查看日志
docker-compose -f build/docker/docker-compose.yml logs -f

# 停止所有服务
docker-compose -f build/docker/docker-compose.yml down

# 停止并移除卷
docker-compose -f build/docker/docker-compose.yml down -v
```

#### 手动分布式设置

如果不使用 Docker Compose：

```bash
# 1. 启动 Redis
docker run -d --name redis -p 6379:6379 redis:7-alpine

# 2. 启动主节点
./build/bin/osmedeus serve --master --port 8001

# 3. 启动工作节点（在同一台或不同机器上）
./build/bin/osmedeus worker join --redis-url redis://localhost:6379
```

#### 提交分布式扫描

```bash
# 提交扫描到分布式队列
./build/bin/osmedeus scan -f general -t example.com -D

# 使用自定义 Redis URL
./build/bin/osmedeus scan -f general -t example.com -D --redis-url redis://redis-host:6379

# 检查工作节点状态
./build/bin/osmedeus worker status
```

## 设置

### 配置文件

默认位置：`~/osmedeus-base/osm-settings.yaml`

```yaml
base_folder: ~/osmedeus-base

environments:
  binaries_path: "{{base_folder}}/binaries"
  data: "{{base_folder}}/data"
  workspaces: ~/workspaces-osmedeus
  workflows: "{{base_folder}}/workflows"

server:
  host: 0.0.0.0
  port: 8001

# 分布式模式必需
redis:
  host: localhost
  port: 6379
  password: ""    # 可选
  db: 0

database:
  db_engine: sqlite    # 或 postgresql
  db_path: "{{base_folder}}/osm-data.db"

client:
  username: admin
  password: admin
  jwt:
    secret: "change-this-in-production"
    expiration_minutes: 60

scan_tactic:
  aggressive: 40
  default: 10
  gently: 5
```

### 环境变量

| 变量               | 描述             | 默认值           |
| ------------------ | ---------------- | ---------------- |
| `REDIS_HOST`       | Redis 主机名     | localhost        |
| `REDIS_PORT`       | Redis 端口       | 6379             |
| `OSM_BASE_FOLDER`  | 基础文件夹路径   | \~/osmedeus-base |

### 命令行覆盖

```bash
# 覆盖基础文件夹
osmedeus -b /custom/path scan -f general -t example.com

# 覆盖工作流文件夹
osmedeus -F /custom/workflows workflow list

# 覆盖 Redis URL（分布式模式）
osmedeus scan -f general -t example.com -D --redis-url redis://user:pass@host:6379/0
```

## Docker Compose 参考

包含的 `build/docker/docker-compose.yml` 提供了完整的分布式设置：

### 服务

| 服务     | 用途                     | 端口 |
| -------- | ------------------------ | ---- |
| `redis`  | 任务队列与协调           | 6379 |
| `master` | API 服务器与任务分发器   | 8001 |
| `worker` | 任务执行器（可扩展）     | -    |

### 卷

| 卷               | 用途                     |
| ---------------- | ------------------------ |
| `redis-data`     | Redis 持久化             |
| `osmedeus-data`  | 工作流与配置             |
| `workspaces`     | 扫描输出数据             |

### 扩展

```bash
# 动态扩展工作节点
docker-compose -f build/docker/docker-compose.yml up -d --scale worker=10

# 查看运行中的容器
docker-compose -f build/docker/docker-compose.yml ps
```

## 生产环境注意事项

### 安全

1. **身份验证**：切勿在生产环境中使用 `-A`（无认证）
2. **JWT Secret**：更改配置中的默认 JWT secret
3. **TLS**：使用反向代理（nginx, traefik）实现 HTTPS
4. **网络**：将 Redis 访问限制在内部网络

```yaml
# 示例：安全的 JWT 配置
client:
  jwt:
    secret: "your-256-bit-secret-key-here"
    expiration_minutes: 30
```

### 资源限制

docker-compose.yml 中的工作节点资源限制：

```yaml
deploy:
  resources:
    limits:
      cpus: '1'
      memory: 1G
    reservations:
      cpus: '0.5'
      memory: 512M
```

根据工作流需求调整。

### 健康检查

Docker 镜像包含内置健康检查：

```bash
# 检查主节点健康
curl http://localhost:8001/health

# 检查就绪状态
curl http://localhost:8001/health/ready
```

### 日志

```bash
# 查看主节点日志
docker logs osmedeus-master -f

# 查看所有工作节点日志
docker-compose -f build/docker/docker-compose.yml logs -f worker

# 日志级别由 --verbose/-v 标志控制
./build/bin/osmedeus -v serve
```

### 数据库选项

对于生产环境，考虑使用 PostgreSQL 替代 SQLite：

```yaml
database:
  db_engine: postgresql
  db_host: postgres-host
  db_port: 5432
  db_name: osmedeus
  db_user: osmedeus
  db_password: secure-password
```

### 备份

```bash
# 备份卷
docker run --rm \
  -v osmedeus-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/osmedeus-backup.tar.gz /data

# 备份工作空间
docker run --rm \
  -v workspaces:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/workspaces-backup.tar.gz /data
```

## Ansible 部署

使用 Ansible 在 Ubuntu/Debian 服务器上部署 Osmedeus。使用官方安装脚本、SQLite 存储，无需 Redis——专为简单的单主机设置设计。

### 先决条件

* **控制机器**：Ansible 2.12+
* **目标服务器**：Ubuntu 20.04+ 或 Debian 11+
* 具有 root 或 sudo 权限的 SSH 访问

### 快速开始

```bash
cd build/infra

# 1. 复制并编辑清单
cp inventory.example.ini inventory.ini
# 编辑 inventory.ini，填入服务器 IP/主机名

# 2. 使用安全凭据部署
ansible-playbook -i inventory.ini deploy.yaml \
  -e osm_admin_password=YourSecurePassword \
  -e osm_jwt_secret=$(openssl rand -base64 32)

# 3. 试运行（预览更改而不应用）
ansible-playbook -i inventory.ini deploy.yaml --check
```

### 执行内容

1. 安装系统依赖（curl, tmux, git, chromium 等）
2. 通过 `curl -fsSL https://www.osmedeus.org/install.sh | bash` 安装 Osmedeus
3. 部署配置为 SQLite（无 Redis）的 `osm-settings.yaml`
4. 运行 `osmedeus health` 验证安装
5. 设置 systemd 服务以实现开机自启

### Playbook 文件

```
build/infra/
├── deploy.yaml                 # 主 playbook
├── inventory.example.ini       # 示例清单
└── templates/
    ├── osm-settings.yaml.j2   # 设置配置模板
    └── osmedeus.service.j2    # Systemd 单元模板
```

### 变量

| 变量                         | 默认值                             | 描述                       |
| ---------------------------- | ---------------------------------- | -------------------------- |
| `osm_server_port`            | `8002`                             | API 服务器端口             |
| `osm_admin_user`             | `admin`                            | 管理员用户名               |
| `osm_admin_password`         | `CHANGE_ME_ADMIN_PASSWORD`         | 管理员密码                 |
| `osm_jwt_secret`             | `CHANGE_ME_JWT_SECRET_MIN_32_CHARS`| JWT 签名密钥               |
| `osm_jwt_expiration_minutes` | `1440`                             | 令牌过期时间（24小时）     |
| `osm_threads_aggressive`     | `50`                               | 激进扫描线程数             |
| `osm_threads_default`        | `20`                               | 默认扫描线程数             |
| `osm_threads_gently`         | `5`                                | 温和扫描线程数             |
| `osm_enable_service`         | `true`                             | 安装 systemd 服务          |
| `osm_telegram_enabled`       | `false`                            | 启用 Telegram 通知         |
| `osm_telegram_bot_token`     | `""`                               | Telegram 机器人令牌         |
| `osm_telegram_chat_id`       | `""`                               | Telegram 聊天 ID           |
| `osm_global_variables`       | `[]`                               | 工作流的额外环境变量       |

### 自定义

在部署时使用 `-e` 覆盖任何变量：

```bash
# 自定义端口和线程设置
ansible-playbook -i inventory.ini deploy.yaml \
  -e osm_server_port=9090 \
  -e osm_threads_default=30

# 带 Telegram 通知
ansible-playbook -i inventory.ini deploy.yaml \
  -e osm_telegram_enabled=true \
  -e osm_telegram_bot_token=your_bot_token \
  -e osm_telegram_chat_id=your_chat_id

# 跳过 systemd 服务设置
ansible-playbook -i inventory.ini deploy.yaml \
  -e osm_enable_service=false
```

### 部署后

```bash
# 检查服务状态
ssh root@YOUR_SERVER systemctl status osmedeus

# 运行扫描
ssh root@YOUR_SERVER osmedeus run -f general -t example.com

# 查看日志
ssh root@YOUR_SERVER journalctl -u osmedeus -f
```

## 故障排除

### 常见问题

**工作节点无法连接：**

```bash
# 检查 Redis 连接
docker exec osmedeus-redis redis-cli ping

# 检查工作节点日志
docker-compose logs worker
```

**扫描未执行：**

```bash
# 验证工作流是否存在
./build/bin/osmedeus workflow list

# 检查主节点日志
docker logs osmedeus-master
```

**端口冲突：**

```bash
# 使用不同端口
docker run -p 8080:8001 osmedeus:latest
```

### 有用命令

```bash
# 环境健康检查
./build/bin/osmedeus health

# 验证工作流
./build/bin/osmedeus workflow validate <workflow-name>

# 测试工作流（试运行）
./build/bin/osmedeus scan -f general -t example.com --dry-run
```
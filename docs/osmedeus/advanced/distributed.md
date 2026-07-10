> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 分布式执行

![分布式执行](https://mintcdn.com/osmedeus/0qu3QQdMjynOzfxl/images/cli/cli-distributed-run.png?fit=max&auto=format&n=0qu3QQdMjynOzfxl&q=85&s=4bbd69e298eea9fa4c1fd95461f30772)

使用 Redis 协调，跨多台机器扩展扫描能力。

## 体系结构

```
+-----------------+     +-----------------+
|     主节点      |     |     Redis       |
|   (API 服务器)  |---->|   (任务队列)    |
+-----------------+     +--------+--------+
                                 |
        +------------------------+------------------------+
        |                        |                        |
        v                        v                        v
+---------------+       +---------------+       +---------------+
|   工作节点 1   |       |   工作节点 2   |       |   工作节点 N   |
+---------------+       +---------------+       +---------------+
```

## 组件

### 主节点

* 用于提交任务的 API 服务器
* 任务分发协调器
* 结果聚合
* 工作节点健康监控

### 工作节点

* 执行分配的任务
* 报告进度和结果
* 失败时自动重连
* 向主节点发送心跳
* 工作节点 ID 格式：`wosm-<uuid8>`（例如 `wosm-a1b2c3d4`）
* 默认别名：当未提供 `--alias` 时，为 `wosm-<public-ip>` 或 `wosm-<local-ip>`

### Redis

* 任务队列存储
* 工作节点注册
* 结果存储
* 事件的发布/订阅

## 设置

### 0. 启动 Redis（如果没有）

```bash
# Docker
docker run -d -p 6379:6379 redis:7-alpine

# 持久化方式
docker run -d -p 6379:6379 \
  -v redis-data:/data \
  redis:7-alpine redis-server --appendonly yes

# 然后测试连接 localhost:6379
```

### 1. 启动主节点

```bash
## 配置 Redis 连接
osmedeus config set redis.host 127.0.0.1
osmedeus config set redis.port 6379

osmedeus server --master
```

或使用自定义 Redis：

```bash
osmedeus server --master --redis-url redis://redis-host:6379
```

你也可以在不启用主节点模式的情况下运行工作节点，只要它们能连接到同一个 Redis 实例，并通过 REST 服务器 API 进行通信。但此时结果不会在主节点 API 中聚合，你需要通过 CLI 手动运行扫描。

### 2. 启动工作节点

```bash
osmedeus worker join --redis-url redis://host.docker.internal:6379
```

启用公网 IP 检测（用于默认别名和 SSH 路由）：

```bash
osmedeus worker join --redis-url redis://host.docker.internal:6379 --get-public-ip
```

使用自定义别名：

```bash
osmedeus worker join --redis-url redis://host.docker.internal:6379 --alias my-worker
```

或者如果主节点在本地：

```bash
## 配置 Redis 连接
osmedeus config set redis.host 127.0.0.1
osmedeus config set redis.port 6379

osmedeus worker join
```

### 3. 提交任务与列出工作节点

```bash
# 通过 CLI
osmedeus run --distributed-run -f general -t hackerone.com

# 通过 API
curl -X POST http://master:8002/osm/api/runs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"flow": "general", "target": "example.com", "distributed": true}'


# 列出工作节点（使用 --json 获取 JSON 输出）
osmedeus worker ls
```

### 4. 在主节点/工作节点上运行实用脚本（可选）

```bash
# 在主节点上运行，以在所有工作节点上执行命令
osmedeus eval "run_on_worker('all', 'bash', 'touch /tmp/on-worker')" --redis-url redis://host.docker.internal:6379

## 在工作节点上运行，以在主节点上执行命令
osmedeus worker --redis-url redis://host.docker.internal:6379 eval "run_on_master('bash', 'touch /tmp/on-master')"
```

***

## 任务分发

### 任务生命周期

```
1. 客户端向主节点提交任务
2. 主节点将任务加入 Redis 队列
3. 可用工作节点认领任务
4. 工作节点执行工作流
5. 工作节点报告进度/结果
6. 主节点聚合结果
7. 通过 API 获取结果
```

### 负载均衡

任务采用拉取模型分发：

* 工作节点轮询可用任务
* 第一个可用的工作节点认领任务
* 无需中央调度

### 任务优先级

（未来功能）任务可设置优先级：

* 高：安全关键扫描
* 正常：常规评估
* 低：后台枚举

## 监控

### 工作节点状态

```bash
# CLI
osmedeus worker status

# API
curl http://master:8002/osm/api/workers \
  -H "Authorization: Bearer $TOKEN"
```

### 任务状态

```bash
# 列出任务
curl http://master:8002/osm/api/tasks \
  -H "Authorization: Bearer $TOKEN"

# 获取任务详情
curl http://master:8002/osm/api/tasks/task-123 \
  -H "Authorization: Bearer $TOKEN"
```

## 远程监控

使用 `client` 命令从任何机器监控分布式运行：

```bash
# 设置连接详情
export OSM_REMOTE_URL=http://master:8080
export OSM_REMOTE_AUTH_KEY=your-api-key

# 监控运行
osmedeus client fetch -t runs --status running --refresh 5s

# 检查特定运行的步骤结果
osmedeus client fetch -t step_results --workspace example_com

# 查看漏洞
osmedeus client fetch -t vulnerabilities --severity critical
```

## Docker Compose 设置

```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: osm-e2e-redis
    ports:
      - "6399:6379"
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - osm-e2e-network

  master:
    build:
      context: ../..
      dockerfile: build/docker/Dockerfile
    image: osmedeus:e2e
    container_name: osm-e2e-master
    ports:
      - "8002:8002"
    volumes:
      - workspaces:/root/workspaces-osmedeus
    depends_on:
      redis:
        condition: service_healthy
    command: ["serve", "--master", "--redis-url", "redis://redis:6379", "-A"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8002/health"]
      interval: 10s
      timeout: 5s
      start_period: 10s
      retries: 5
    networks:
      - osm-e2e-network

  worker:
    image: osmedeus:e2e
    container_name: osm-e2e-worker
    volumes:
      - workspaces:/root/workspaces-osmedeus
    depends_on:
      redis:
        condition: service_healthy
      master:
        condition: service_healthy
    command: ["worker", "join", "--redis-url", "redis://redis:6379"]
    networks:
      - osm-e2e-network

volumes:
  workspaces:
    driver: local

networks:
  osm-e2e-network:
    driver: bridge
```

运行：

```bash
# 分布式端到端堆栈：Redis + 主节点 + 工作节点
# 用法：

make distributed-e2e-up      # 构建镜像 + 启动堆栈 + 等待健康检查
make distributed-e2e-run     # 提交扫描 + 跟踪工作节点日志
make distributed-e2e-down    # 拆除所有组件

```
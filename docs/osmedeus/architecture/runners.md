# Runners

> 命令和步骤的执行环境

Runner 在不同的环境中执行命令。

## Runner 类型

| Runner   | 描述               | 用例                               |
| -------- | ------------------ | ---------------------------------- |
| `host`   | 本地机器执行       | 默认，快速，无隔离                  |
| `docker` | 容器执行           | 隔离、可重现、工具打包              |
| `ssh`    | 远程机器执行       | 分布式扫描、远程资源                |

## Host Runner

使用 shell 在本地机器上执行命令。

### 配置

```yaml
kind: module
name: local-scan
runner: host        # 默认值，可省略

steps:
  - name: scan
    type: bash
    command: nmap -sV {{target}}
```

### 特性

* 使用 `sh -c` 执行命令
* 继承 Osmedeus 进程的环境
* 步骤之间无隔离
* 执行速度最快

## Docker Runner

在 Docker 容器内执行命令。

### 模块级配置

```yaml
kind: module
name: docker-scan
runner: docker
runner_config:
  image: projectdiscovery/nuclei:latest
  volumes:
    - "{{Output}}:/output"
  environment:
    - "API_KEY={{api_key}}"
  persistent: false    # 默认：临时容器

steps:
  - name: scan
    type: bash
    command: nuclei -u {{target}} -o /output/nuclei.txt
```

### Runner 配置选项

| 选项          | 类型        | 描述                              |
| ------------- | ----------- | --------------------------------- |
| `image`       | string      | Docker 镜像（必需）               |
| `volumes`     | \[]string   | 卷挂载（`主机:容器`）             |
| `environment` | \[]string   | 环境变量                          |
| `persistent`  | bool        | 在步骤之间保持容器运行            |
| `network`     | string      | Docker 网络名称                   |
| `extra_args`  | \[]string   | 额外的 docker run 参数            |

### 执行模式

**临时（默认）**：每个步骤运行 `docker run --rm`

```yaml
runner_config:
  image: alpine:latest
  persistent: false    # 每个步骤新建容器
```

**持久化**：容器保持运行，步骤使用 `docker exec`

```yaml
runner_config:
  image: alpine:latest
  persistent: true     # 复用容器
```

### 按步骤使用 Docker（remote-bash）

在不设置模块级 runner 的情况下，为特定步骤使用 Docker：

```yaml
kind: module
name: hybrid
runner: host

steps:
  - name: local-step
    type: bash
    command: echo "Running locally"

  - name: docker-step
    type: remote-bash
    step_runner: docker
    step_runner_config:
      image: alpine:latest
      volumes:
        - "{{Output}}:/output"
    command: cat /etc/os-release
```

## SSH Runner

通过 SSH 在远程机器上执行命令。

### 模块级配置

```yaml
kind: module
name: remote-scan
runner: ssh
runner_config:
  host: scanner.example.com
  port: 22
  user: scanner
  key_file: ~/.ssh/scanner_key
  # 或
  password: secret     # 不推荐

steps:
  - name: scan
    type: bash
    command: nmap -sV {{target}}
```

### Runner 配置选项

| 选项           | 类型   | 描述                    |
| -------------- | ------ | ----------------------- |
| `host`         | string | SSH 主机名（必需）      |
| `port`         | int    | SSH 端口（默认：22）    |
| `user`         | string | SSH 用户名（必需）      |
| `key_file`     | string | 私钥路径                |
| `password`     | string | SSH 密码（安全性较低）  |
| `known_hosts`  | string | known_hosts 文件路径    |

### 按步骤使用 SSH（remote-bash）

为特定步骤使用 SSH：

```yaml
kind: module
name: hybrid
runner: host

steps:
  - name: local-prep
    type: bash
    command: echo {{target}} > /tmp/target.txt

  - name: remote-scan
    type: remote-bash
    step_runner: ssh
    step_runner_config:
      host: "{{ssh_host}}"
      port: 22
      user: "{{ssh_user}}"
      key_file: ~/.ssh/id_rsa
    command: nmap -sV {{target}}
```

### 文件传输

将文件从远程复制到本地：

```yaml
- name: remote-scan
  type: remote-bash
  step_runner: ssh
  step_runner_config:
    host: scanner.example.com
    user: scanner
    key_file: ~/.ssh/key
  command: nmap -sV {{target}} -oN /tmp/result.txt
  step_remote_file: /tmp/result.txt
  host_output_file: "{{Output}}/nmap-result.txt"
```

## Runner 接口

所有 Runner 实现以下接口：

```go
type Runner interface {
    Execute(ctx context.Context, command string) (*CommandResult, error)
    Setup(ctx context.Context) error
    Cleanup(ctx context.Context) error
    Type() core.RunnerType
    IsRemote() bool
}

type CommandResult struct {
    Output   string
    ExitCode int
    Error    error
}
```

## 生命周期

```
1. Setup()    - 初始化 runner（连接 SSH、启动容器）
2. Execute()  - 运行命令（每个步骤调用）
3. Cleanup()  - 清理（断开连接、移除容器）
```

## 选择 Runner

| 场景                     | 推荐 Runner              |
| ------------------------ | ------------------------ |
| 简单的本地扫描           | `host`                   |
| 工具隔离                 | `docker`                 |
| 可重现构建               | `docker`                 |
| 带有工具的远程服务器     | `ssh`                    |
| 分布式扫描               | `ssh` 或分布式模式       |
| 混合环境                 | 按步骤使用 `remote-bash` |

## 最佳实践

### Docker

1. **使用具体的镜像标签**
   ```yaml
   image: projectdiscovery/nuclei:v2.9.0  # 良好
   image: projectdiscovery/nuclei:latest  # 可预测性较差
   ```

2. **仅挂载必要的卷**
   ```yaml
   volumes:
     - "{{Output}}:/output:rw"
     - "{{Data}}/templates:/templates:ro"
   ```

3. **多步骤时使用持久化模式**
   ```yaml
   runner_config:
     persistent: true  # 多步骤工作流更快
   ```

### SSH

1. **使用密钥认证**
   ```yaml
   key_file: ~/.ssh/scanner_key
   # 避免：password: secret
   ```

2. **参数化主机详情**
   ```yaml
   runner_config:
     host: "{{ssh_host}}"
     user: "{{ssh_user}}"
   ```

3. **检查远程工具可用性**
   ```yaml
   - name: check-tools
     type: bash
     command: which nmap nuclei httpx
   ```

### remote-bash

1. **用于混合工作流**
   * 本地文件准备
   * 远程重型扫描
   * 本地结果处理

2. **将结果传回**
   ```yaml
   step_remote_file: /remote/output.txt
   host_output_file: "{{Output}}/output.txt"
   ```

## 下一步

* [Step Types](../workflows/step-types) - 使用 remote-bash
* [Deployment](../getting-started/deployment) - 分布式模式
* [Extending Runners](../extending/runners) - 自定义 Runner
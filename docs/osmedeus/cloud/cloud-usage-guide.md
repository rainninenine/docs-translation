# Cloud 使用指南

Osmedeus Cloud 跨云供应商预配虚拟机，并在其上运行安全工作流或任意命令。本指南涵盖架构、配置和操作模式。

## 工作原理

```
本地机器                          云供应商
┌──────────────┐                      ┌──────────────────┐
│ osmedeus     │   1. 预配            │  工作节点 VM 1   │
│ cloud run    │ ──────────────────►  │  ┌────────────┐  │
│              │   2. SSH 设置        │  │ osmedeus   │  │
│              │ ──────────────────►  │  │ + 工具     │  │
│              │   3. 流式输出        │  └────────────┘  │
│              │ ◄──────────────────  │                  │
│              │                      ├──────────────────┤
│              │   （每个节点相同）    │  工作节点 VM 2   │
│              │ ◄──────────────────► │  ...             │
│              │                      ├──────────────────┤
│              │   4. 同步结果        │  工作节点 VM N   │
│              │ ◄──────────────────  │  ...             │
│              │   5. 销毁            └──────────────────┘
└──────────────┘
```

**生命周期：**

1. **预配** -- 通过 Pulumi 创建 VM（或复用现有 VM）
2. **设置** -- SSH 进入每个工作节点，运行设置命令（安装 osmedeus、工具等）
3. **执行** -- 运行工作流或自定义命令，实时流式输出
4. **同步** -- 将结果下载到本地机器（可选）
5. **销毁** -- 拆除基础设施（可选，可自动进行）

## 支持的供应商

| 供应商 | 配置键 | 实例类型 |
|----------|-----------|----------------|
| AWS | `aws` | t3.medium、t3.large、t3.xlarge |
| DigitalOcean | `digitalocean` | s-2vcpu-4gb、s-4vcpu-8gb、s-8vcpu-16gb |
| GCP | `gcp` | n1-standard-2、n1-standard-4 |
| Hetzner | `hetzner` | cx22、cx32、cx42 |
| Linode | `linode` | g6-standard-2、g6-standard-4 |
| Azure | `azure` | Standard_B2s、Standard_D2s_v3 |

## 配置

Cloud 配置位于 `~/.osmedeus/cloud/cloud-settings.yaml`。使用以下命令管理：

```bash
# 设置值
osmedeus cloud config set <key> <value>

# 查看当前配置
osmedeus cloud config list

# 重置为默认值
osmedeus cloud config clean
```

### 必需配置

每个供应商需要四样东西：**启用云功能**、**凭据**、**SSH 密钥**和**设置命令**。

```bash
# 0. 启用云功能
osmedeus config set cloud.enabled true

# 1. 供应商凭据（示例：AWS）
osmedeus cloud config set providers.aws.access_key_id ${AWS_ACCESS_KEY_ID}
osmedeus cloud config set providers.aws.secret_access_key ${AWS_SECRET_ACCESS_KEY}
osmedeus cloud config set providers.aws.region ap-southeast-1

# 2. SSH 密钥（用于连接到工作节点）
osmedeus cloud config set ssh.private_key_path ~/.ssh/id_rsa
osmedeus cloud config set ssh.public_key_path ~/.ssh/id_rsa.pub

# 3. 先清空设置脚本，然后添加设置命令（在每个工作节点扫描前运行）
osmedeus cloud config set setup.commands.clear ""
osmedeus cloud config set setup.commands.add "curl -fsSL https://www.osmedeus.org/install.sh | bash"
osmedeus cloud config set setup.commands.add "osmedeus install base --preset"

# 4. 设置默认供应商
osmedeus cloud config set defaults.provider aws
```

### 可选配置

```bash
# 实例类型
osmedeus cloud config set providers.aws.instance_type t3.large

# 使用竞价/抢占式实例（便宜 70-80%）
osmedeus cloud config set providers.aws.use_spot true

# 成本限制
osmedeus cloud config set limits.max_hourly_spend 1.00
osmedeus cloud config set limits.max_total_spend 10.00
osmedeus cloud config set limits.max_instances 10

# 默认超时
osmedeus cloud config set defaults.timeout 2h

# SSH 用户（大多数供应商默认 root，AWS 默认 ubuntu）
osmedeus cloud config set ssh.user root
```

### 后置设置命令

后置设置命令在主设置之后在每个工作节点上运行，模板变量会被展开：

```bash
osmedeus cloud config set setup.post_commands.add "echo 'Worker {{index}} ready at {{public_ip}}'"
```

可用变量：`{{public_ip}}`、`{{private_ip}}`、`{{worker_name}}`、`{{worker_id}}`、`{{infra_id}}`、`{{provider}}`、`{{ssh_user}}`、`{{index}}`

## 两种执行模式

### 工作流模式（默认）

在远程工作节点上运行 osmedeus flow 或 module：

```bash
# 运行 flow
osmedeus cloud run -f fast -t example.com

# 运行 module
osmedeus cloud run -m enum-subdomain -t example.com
```

### 自定义命令模式

在远程工作节点上运行任意 Shell 命令——无需 osmedeus 工作流：

```bash
osmedeus cloud run --custom-cmd "nmap -sV {{Target}} -oA /tmp/osm-custom/nmap" -t example.com
```

`--custom-cmd` 与 `-f`/`-m` 互斥。详见下方的 [自定义命令模式](#自定义命令模式详情)。

## 基础设施管理

### 预配

```bash
# 使用 cloud run 预配（创建 + 运行 + 可选销毁）
osmedeus cloud run -f fast -t example.com --instances 3

# 单独预配（不扫描）
osmedeus cloud create --provider aws -n 3
```

### 列出

```bash
osmedeus cloud list
```

### 复用现有基础设施

```bash
# 从保存的状态自动发现
osmedeus cloud run -f fast -t example.com --reuse

# 直接指定 IP
osmedeus cloud run -f fast -t example.com --reuse-with "1.2.3.4,5.6.7.8"
```

### 销毁

```bash
# 销毁特定基础设施
osmedeus cloud destroy <infra-id>

# 销毁所有
osmedeus cloud destroy all --force
```

## 目标分发

当在多个工作节点上扫描多个目标时，osmedeus 将目标列表分成块：

```bash
# 100 个目标分配到 5 个工作节点 = 每个 20 个
osmedeus cloud run -f fast -T targets.txt --instances 5

# 控制块大小：每个工作节点 10 个目标
osmedeus cloud run -f fast -T targets.txt --instances 10 --chunk-size 10

# 控制块数量：精确分成 3 块
osmedeus cloud run -f fast -T targets.txt --instances 5 --chunk-count 3
```

每个工作节点在远程机器上收到其块作为文件，路径为 `/tmp/osm-targets-{i}.txt`。

## 自定义命令模式详情

在云实例上运行任意命令，无需使用 osmedeus 工作流。命令在远程的 `/tmp/osm-custom/` 中运行。

### 标志

| 标志 | 描述 |
|------|-------------|
| `--custom-cmd` | 要运行的命令（可重复，每个工作节点顺序执行） |
| `--custom-post-cmd` | 在所有自定义命令成功后运行（可重复） |
| `--sync-path` | 执行后要下载的远程路径（可重复） |
| `--sync-dest` | 下载的本地基础目录（默认：`./osm-sync-back`） |

### 模板变量

所有命令和同步路径支持以下变量：

| 变量 | 描述 | 示例 |
|----------|-------------|---------|
| `{{Target}}` | 目标字符串，或使用 `-T` 时的块文件路径 | `example.com` 或 `/tmp/osm-targets-0.txt` |
| `{{public_ip}}` | 工作节点的公网 IP | `203.0.113.10` |
| `{{private_ip}}` | 工作节点的私网 IP | `10.0.0.5` |
| `{{worker_name}}` | 资源名称 | `osmw-1775159841-0` |
| `{{worker_id}}` | 云资源 ID | `i-0437adf5...` |
| `{{infra_id}}` | 基础设施 ID | `cloud-aws-1775159841` |
| `{{provider}}` | 供应商名称 | `aws` |
| `{{ssh_user}}` | SSH 用户名 | `ubuntu` |
| `{{index}}` | 工作节点索引 | `0`、`1`、`2` |

### 执行规则

- 自定义命令在每个工作节点上**顺序**执行，但跨工作节点**并行**执行
- 如果任何 `--custom-cmd` 失败（非零退出），该工作节点的剩余命令和所有 `--custom-post-cmd` 将被跳过
- 后置命令失败会被记录，但不影响其他工作节点

### 同步回传

下载的文件放置在：`<sync-dest>/<worker_name>-<ip>/<remote_path>`

例如，从 IP 为 `1.2.3.4` 的工作节点 `osmw-0` 执行 `--sync-path /tmp/osm-custom/results.txt`：
```
./osm-sync-back/osmw-0-1.2.3.4/tmp/osm-custom/results.txt
```

### 示例

```bash
# 简单：在云实例上运行 nmap
osmedeus cloud run \
  --custom-cmd "nmap -sV {{Target}} -oA /tmp/osm-custom/nmap-result" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy

# 多步骤管道带后处理
osmedeus cloud run \
  --custom-cmd "subfinder -d {{Target}} -o /tmp/osm-custom/subs.txt" \
  --custom-cmd "cat /tmp/osm-custom/subs.txt | httpx -o /tmp/osm-custom/live.txt" \
  --custom-post-cmd "wc -l /tmp/osm-custom/live.txt" \
  --sync-path "/tmp/osm-custom/subs.txt" \
  --sync-path "/tmp/osm-custom/live.txt" \
  -t example.com

# 将目标列表分发到 5 个工作节点
osmedeus cloud run \
  --custom-cmd "cat {{Target}} | nuclei -o /tmp/osm-custom/nuclei.txt" \
  --sync-path "/tmp/osm-custom/nuclei.txt" \
  --sync-dest "./nuclei-results" \
  -T targets.txt --instances 5 --auto-destroy
```

## 同步结果

### 工作流模式：`--sync-back`

从远程工作节点导出 osmedeus 项目空间（包括数据库状态）并在本地导入：

```bash
osmedeus cloud run -f fast -t example.com --sync-back
```

### 自定义模式：`--sync-path`

通过 SFTP 下载特定文件或目录：

```bash
osmedeus cloud run --custom-cmd "..." --sync-path "/tmp/osm-custom/" -t example.com
```

## 成本管理

### 预配前估算

预配前会估算成本。设置限制以防止超支：

```bash
osmedeus cloud config set limits.max_hourly_spend 1.00
osmedeus cloud config set limits.max_total_spend 10.00
osmedeus cloud config set limits.max_instances 10
```

### 竞价/抢占式实例

节省 70-80% 的实例成本：

```bash
# AWS 竞价实例
osmedeus cloud config set providers.aws.use_spot true

# GCP 抢占式实例
osmedeus cloud config set providers.gcp.use_preemptible true
```

### 成本参考

| 供应商 | 实例 | vCPU | RAM | 每小时 |
|----------|----------|------|-----|--------|
| Hetzner | cx22 | 2 | 4 GB | ~$0.007 |
| Linode | g6-standard-2 | 2 | 4 GB | $0.018 |
| DigitalOcean | s-2vcpu-4gb | 2 | 4 GB | $0.02232 |
| AWS | t3.medium | 2 | 4 GB | $0.0416 |
| GCP | n1-standard-2 | 2 | 7.5 GB | $0.095 |
| Azure | Standard_B2s | 2 | 4 GB | $0.042 |

**示例：** 5 个 DigitalOcean s-2vcpu-4gb 实例运行 2 小时 = 5 x $0.02232 x 2 = **$0.22**

## 工作节点设置

预配后通过 SSH 设置工作节点。设置流程：

1. **Cloud-init**（自动）：安装 SSH 密钥、基础包
2. **设置命令**（`setup.commands`）：安装 osmedeus、工具、基础数据
3. **后置设置命令**（`setup.post_commands`）：带模板变量的每个工作节点配置

### Ansible 替代方案

对于复杂设置，使用 Ansible 代替 SSH 命令：

```bash
osmedeus cloud config set setup.ansible.enabled true
osmedeus cloud config set setup.ansible.playbook_path /path/to/playbook.yaml
osmedeus cloud run -f fast -t example.com --ansible
```

### 在现有机器上设置

```bash
osmedeus cloud setup --reuse-with "1.2.3.4,5.6.7.8"
```

## 故障排除

### 工作节点无法连接

```bash
# 详细设置以查看 SSH 输出
osmedeus cloud run -f fast -t example.com --verbose-setup

# 调试模式以获取完整日志
osmedeus cloud run -f fast -t example.com --debug
```

### 基础设施卡住

```bash
# 列出所有基础设施
osmedeus cloud list

# 强制销毁所有
osmedeus cloud destroy all --force
```

### 超出成本

如果达到成本限制，预配将被阻止。调整限制：

```bash
osmedeus cloud config set limits.max_hourly_spend 5.00
```

## 最佳实践

1. **在运行大规模扫描前始终设置成本限制**
2. **使用 `--auto-destroy`** 避免遗忘实例产生费用
3. **对非关键扫描使用竞价实例**（节省 70-80%）
4. **使用 `--reuse`** 避免重复预配，适合迭代工作
5. **从小规模开始**——先用 1 个实例测试，再扩展
6. **使用预装工具的自定义快照**，将设置时间从 5 分钟降至 30 秒
7. **定期检查 `cloud list`** 以验证没有孤儿基础设施

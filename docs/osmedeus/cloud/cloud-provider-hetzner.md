# Hetzner 供应商指南

在 Hetzner Cloud 服务器上运行 osmedeus cloud 的分步指南。Hetzner 在所有支持的供应商中提供最低的单实例成本，非常适合高容量扫描。

## 前提条件

- 一个 Hetzner Cloud 账户（https://console.hetzner.cloud）
- 一个 API 令牌
- 一个 SSH 密钥对（本地 `~/.ssh/id_rsa` 和 `~/.ssh/id_rsa.pub`）

### 获取 API 令牌

1. 前往 **Hetzner Cloud Console** > 选择您的项目（或创建一个）
2. **安全** > **API 令牌** > **生成 API 令牌**
3. 将权限设置为 **读写**
4. 复制令牌（仅显示一次）

您也可以将其存储为环境变量：

```bash
export HETZNER_API_TOKEN="your-token-here"
```

## 配置

### 最小设置

```bash
# 启用云功能
osmedeus config set cloud.enabled true

# 凭据
osmedeus cloud config set providers.hetzner.token ${HETZNER_API_TOKEN}
osmedeus cloud config set providers.hetzner.location hel1
osmedeus cloud config set providers.hetzner.server_type "cx23"  # 2 vCPU，4GB RAM（当前代）
osmedeus cloud config set defaults.provider hetzner

# SSH
osmedeus cloud config set ssh.private_key_path ~/.ssh/id_rsa
osmedeus cloud config set ssh.public_key_path ~/.ssh/id_rsa.pub
osmedeus cloud config set ssh.user root

# 先清空设置脚本
osmedeus cloud config set setup.commands.clear ""

# 工作节点设置
osmedeus cloud config set setup.commands.add "sudo apt-get update"
osmedeus cloud config set setup.commands.add "sudo apt-get install -y -qq curl git tmux unzip jq rsync"
osmedeus cloud config set setup.commands.add "curl -fsSL https://www.osmedeus.org/install.sh | bash"
osmedeus cloud config set setup.commands.add "osmedeus install base --preset"
```

### 服务器类型

Hetzner 的定价明显低于其他供应商：

| 服务器类型 | vCPU | RAM | 磁盘 | $/时 | $/月 | 最佳用途 |
|-------------|------|-----|------|------|---------|----------|
| cx22 | 2 | 4 GB | 40 GB | ~$0.007 | ~$4.50 | 轻量扫描，单目标 |
| cx32 | 4 | 8 GB | 80 GB | ~$0.013 | ~$8.50 | 通用扫描 |
| cx42 | 8 | 16 GB | 160 GB | ~$0.025 | ~$16.50 | 重型扫描，并行工具 |
| cx52 | 16 | 32 GB | 320 GB | ~$0.050 | ~$33.00 | 大规模操作 |
| cpx21 | 3 | 4 GB | 80 GB | ~$0.008 | ~$5.50 | CPU 优化扫描 |
| cpx31 | 4 | 8 GB | 160 GB | ~$0.015 | ~$10.00 | CPU 优化，更多 RAM |

```bash
osmedeus cloud config set providers.hetzner.server_type cx32
```

### 位置

| 位置 | 代码 | 区域 |
|----------|------|--------|
| 法尔肯斯坦 | `fsn1` | 德国 |
| 纽伦堡 | `nbg1` | 德国 |
| 赫尔辛基 | `hel1` | 芬兰 |
| 阿什本 | `ash` | 美国东部 |
| 希尔斯伯勒 | `hil` | 美国西部 |
| 新加坡 | `sin` | 亚洲 |

```bash
osmedeus cloud config set providers.hetzner.location fsn1
```

### 自定义镜像

使用预构建的快照以加快启动速度：

```bash
# 手动设置服务器并安装所有工具后：
# Hetzner Console > Servers > your-server > Snapshots > Create Snapshot
# 记下快照 ID

osmedeus cloud config set providers.hetzner.image 12345678
```

### SSH 密钥（可选）

如果您在 Hetzner Cloud 中注册了 SSH 密钥：

```bash
# Hetzner Console > Security > SSH Keys > 记下密钥名称
osmedeus cloud config set providers.hetzner.ssh_key_name my-key-name
```

### 成本限制

```bash
osmedeus cloud config set limits.max_hourly_spend 0.50
osmedeus cloud config set limits.max_total_spend 5.00
osmedeus cloud config set limits.max_instances 20
```

## 示例

### 快速域名侦察

```bash
osmedeus cloud run -f fast -t example.com --auto-destroy
```

成本：~$0.007（1 x cx22 x 1 小时）——不到 1 美分。

### 预算批量扫描

Hetzner 的低价使其非常适合扫描大量目标：

```bash
# 20 个工作节点扫描 200 个目标，约 $0.28
osmedeus cloud run \
  -f fast -T targets.txt --instances 20 \
  --sync-back --auto-destroy
```

成本：20 x $0.007 x 2 小时 = **$0.28**

### 自定义命令管道

```bash
osmedeus cloud run \
  --custom-cmd "subfinder -d {{Target}} -o /tmp/osm-custom/subs.txt" \
  --custom-cmd "cat /tmp/osm-custom/subs.txt | httpx -o /tmp/osm-custom/live.txt" \
  --custom-cmd "cat /tmp/osm-custom/live.txt | nuclei -o /tmp/osm-custom/nuclei.txt" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy
```

成本：~$0.007

### 大规模分布式 Nuclei

```bash
# 将 10,000 个 URL 分配到 10 个 Hetzner 工作节点
osmedeus cloud run \
  --custom-cmd "cat {{Target}} | nuclei -o /tmp/osm-custom/results.txt" \
  --sync-path "/tmp/osm-custom/results.txt" \
  --sync-dest "./nuclei-hetzner" \
  -T urls.txt --instances 10 --auto-destroy
```

成本：10 x $0.007 x 1 小时 = **$0.07**

### 端口扫描

```bash
# 对 masscan 使用更大的实例（需要更多资源）
osmedeus cloud config set providers.hetzner.server_type cx32

osmedeus cloud run \
  --custom-cmd "masscan {{Target}} -p1-65535 --rate 10000 -oG /tmp/osm-custom/masscan.txt" \
  --custom-cmd "cat /tmp/osm-custom/masscan.txt | grep 'Host:' | awk '{print \\$2\\\":\\\" \\$5}' | sed 's|/.*||' > /tmp/osm-custom/open-ports.txt" \
  --sync-path "/tmp/osm-custom/" \
  -t 203.0.113.0/24 --auto-destroy
```

### 持久低成本实验室

```bash
# 创建 5 个工作节点并全天保持运行
osmedeus cloud create --provider hetzner -n 5

# 全天运行多次扫描
osmedeus cloud run -f fast -t target1.com --reuse
osmedeus cloud run --custom-cmd "nmap -sV {{Target}}" -t target2.com --reuse
osmedeus cloud run -f general -T targets.txt --reuse

# 一天结束时销毁
osmedeus cloud destroy all --force
```

成本：5 x $0.007 x 8 小时 = **$0.28**，5 台机器全天扫描。

### 基于欧洲的扫描

当需要从欧洲 IP 空间发起扫描时，Hetzner 的欧洲位置非常有用：

```bash
# 使用德国数据中心
osmedeus cloud config set providers.hetzner.location fsn1
osmedeus cloud run -f fast -t eu-target.com --auto-destroy

# 使用芬兰数据中心
osmedeus cloud config set providers.hetzner.location hel1
osmedeus cloud run -f fast -t nordic-target.com --auto-destroy
```

### 使用 cx42 的高性能扫描

```bash
# 使用 8 vCPU / 16 GB RAM 实例进行重型并行扫描
osmedeus cloud config set providers.hetzner.server_type cx42

osmedeus cloud run \
  --custom-cmd "subfinder -d {{Target}} -all -o /tmp/osm-custom/subs.txt" \
  --custom-cmd "cat /tmp/osm-custom/subs.txt | httpx -td -threads 200 -o /tmp/osm-custom/live.txt" \
  --custom-cmd "cat /tmp/osm-custom/live.txt | nuclei -c 100 -o /tmp/osm-custom/nuclei.txt" \
  --custom-cmd "cat /tmp/osm-custom/live.txt | katana -d 3 -jc -o /tmp/osm-custom/crawl.txt" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy
```

成本：~$0.025 每小时

## 成本对比

为什么 Hetzner 是批量扫描最便宜的选择：

| 场景 | Hetzner (cx22) | DigitalOcean (s-2vcpu-4gb) | AWS (t3.medium) |
|----------|---------------|---------------------------|-----------------|
| 1 实例 x 1 小时 | $0.007 | $0.022 | $0.042 |
| 5 实例 x 2 小时 | $0.07 | $0.22 | $0.42 |
| 10 实例 x 4 小时 | $0.28 | $0.89 | $1.66 |
| 20 实例 x 8 小时 | $1.12 | $3.57 | $6.66 |

同等规格下，Hetzner 比 DigitalOcean 便宜约 3 倍，比 AWS 便宜约 6 倍。

## 故障排除

### "Unauthorized" 错误

您的 API 令牌无效或已过期。在 Hetzner Cloud Console 中生成新令牌。

```bash
osmedeus cloud config set providers.hetzner.token <new-token>
```

### 服务器类型不可用

某些服务器类型可能在某些位置不可用。尝试不同的位置：

```bash
osmedeus cloud config set providers.hetzner.location nbg1
```

### SSH 连接问题

Hetzner 服务器默认使用 `root` 用户：

```bash
osmedeus cloud config set ssh.user root
```

验证您的 SSH 密钥配置正确：

```bash
osmedeus cloud run --custom-cmd "whoami" -t test --verbose-setup
```

### 速率限制

Hetzner 的 API 有速率限制。如果同时创建多个实例，可能会达到限制。请错开创建时间或联系 Hetzner 支持以增加限制。

### 清理

```bash
# 列出所有基础设施
osmedeus cloud list

# 销毁特定资源
osmedeus cloud destroy <infra-id>

# 销毁所有资源
osmedeus cloud destroy all --force

# 如果状态不同步，直接检查 Hetzner Console：
# Console > Servers > 查找 osmedeus 前缀的服务器
```

## 最佳实践

1. **默认使用 cx22** -- 2 vCPU / 4 GB 足以满足大多数扫描，仅 $0.007/时
2. **水平扩展** -- 对于可并行化的工作负载，10 x cx22 比 1 x cx52 更便宜且更快
3. **始终使用 `--auto-destroy`** -- 即使 $0.007/时，遗忘的实例也会累积成本
4. **使用欧洲位置**（fsn1、nbg1）以获得 Hetzner 网络的最低延迟
5. **预构建快照**用于常用工具配置，以跳过设置时间
6. **设置适度的成本限制** -- 即使 $5.00 的 max_total_spend 在 Hetzner 的定价下也能走得很远

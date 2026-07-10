> ## Documentation Index
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# Hetzner Provider Guide

> 在 Hetzner Cloud 服务器上运行 osmedeus cloud 的分步指南，这是成本最低的供应商

<Frame>
  <img src="https://mintcdn.com/osmedeus/-AJASvFBPI8cnc5L/images/cloud/osm-cloud-setup.png?fit=max&auto=format&n=-AJASvFBPI8cnc5L&q=85&s=981e344fe377fdfc77837a355d086bc6" alt="web-ui-assets" width="3824" height="1296" data-path="images/cloud/osm-cloud-setup.png" />
</Frame>

<Frame>
  <img src="https://mintcdn.com/osmedeus/-AJASvFBPI8cnc5L/images/cloud/osm-cloud-manage-instance.png?fit=max&auto=format&n=-AJASvFBPI8cnc5L&q=85&s=e5defc4a4b47b01a58c36c01793d91ab" alt="Introduction Cloud" width="3824" height="1780" data-path="images/cloud/osm-cloud-manage-instance.png" />
</Frame>

在 Hetzner Cloud 服务器上运行 osmedeus cloud 的分步指南。Hetzner 在所有支持的供应商中提供最低的每实例成本，非常适合大规模扫描。

## Prerequisites

* 一个 Hetzner Cloud 账户（[https://console.hetzner.cloud](https://console.hetzner.cloud)）
* 一个 API 令牌
* 一个 SSH 密钥对（本地 `~/.ssh/id_rsa` 和 `~/.ssh/id_rsa.pub`）

### 获取您的 API 令牌

1. 前往 **Hetzner Cloud Console** > 选择您的项目（或创建一个）
2. **Security** > **API Tokens** > **Generate API Token**
3. 将权限设置为 **Read & Write**
4. 复制令牌（仅显示一次）

您也可以将其存储为环境变量：

```bash theme={null}
export HETZNER_API_TOKEN="your-token-here"
```

## Configuration

### 最小化设置

```bash theme={null}
# 启用云功能
osmedeus config set cloud.enabled true

# 凭据
osmedeus cloud config set providers.hetzner.token ${HETZNER_API_TOKEN}
osmedeus cloud config set providers.hetzner.location hel1
osmedeus cloud config set providers.hetzner.server_type "cx23"  # 2 vCPU, 4GB RAM（当前代）
osmedeus cloud config set defaults.provider hetzner

# SSH
osmedeus cloud config set ssh.private_key_path ~/.ssh/id_rsa
osmedeus cloud config set ssh.public_key_path ~/.ssh/id_rsa.pub
osmedeus cloud config set ssh.user root

# 先清理设置脚本
osmedeus cloud config set setup.commands.clear ""

# Worker 设置
osmedeus cloud config set setup.commands.add "sudo apt-get update"
osmedeus cloud config set setup.commands.add "sudo apt-get install -y -qq curl git tmux unzip jq rsync"
osmedeus cloud config set setup.commands.add "curl -fsSL https://www.osmedeus.org/install.sh | bash"
osmedeus cloud config set setup.commands.add "osmedeus install base --preset"
```

### 服务器类型

Hetzner 的定价显著低于其他供应商：

| 服务器类型 | vCPU | RAM   | 磁盘   | \$/小时     | \$/月      | 最佳用途                    |
| ----------- | ---- | ----- | ------ | --------- | --------- | --------------------------- |
| cx22        | 2    | 4 GB  | 40 GB  | \~\$0.007 | \~\$4.50  | 轻量扫描，单个目标 |
| cx32        | 4    | 8 GB  | 80 GB  | \~\$0.013 | \~\$8.50  | 通用扫描            |
| cx42        | 8    | 16 GB | 160 GB | \~\$0.025 | \~\$16.50 | 重型扫描，并行工具 |
| cx52        | 16   | 32 GB | 320 GB | \~\$0.050 | \~\$33.00 | 大规模操作      |
| cpx21       | 3    | 4 GB  | 80 GB  | \~\$0.008 | \~\$5.50  | CPU 优化扫描      |
| cpx31       | 4    | 8 GB  | 160 GB | \~\$0.015 | \~\$10.00 | CPU 优化，更多 RAM     |

```bash theme={null}
osmedeus cloud config set providers.hetzner.server_type cx32
```

### 位置

| 位置        | 代码   | 区域  |
| ----------- | ------ | ------- |
| Falkenstein | `fsn1` | 德国 |
| Nuremberg   | `nbg1` | 德国 |
| Helsinki    | `hel1` | 芬兰 |
| Ashburn     | `ash`  | 美国东部 |
| Hillsboro   | `hil`  | 美国西部 |
| Singapore   | `sin`  | 亚洲    |

```bash theme={null}
osmedeus cloud config set providers.hetzner.location fsn1
```

### 自定义镜像

使用预构建的快照以加快启动速度：

```bash theme={null}
# 手动设置服务器并安装所有工具后：
# Hetzner Console > Servers > your-server > Snapshots > Create Snapshot
# 记下快照 ID

osmedeus cloud config set providers.hetzner.image 12345678
```

### SSH 密钥（可选）

如果您在 Hetzner Cloud 中注册了 SSH 密钥：

```bash theme={null}
# Hetzner Console > Security > SSH Keys > 记下密钥名称
osmedeus cloud config set providers.hetzner.ssh_key_name my-key-name
```

### 成本限制

```bash theme={null}
osmedeus cloud config set limits.max_hourly_spend 0.50
osmedeus cloud config set limits.max_total_spend 5.00
osmedeus cloud config set limits.max_instances 20
```

## Examples

### 快速域名侦察

```bash theme={null}
osmedeus cloud run -f fast -t example.com --auto-destroy
```

成本：\~\$0.007（1 x cx22 x 1 小时）——不到 1 美分。

### 预算批量扫描

Hetzner 的低价格使其非常适合扫描大量目标：

```bash theme={null}
# 20 个 worker 扫描 200 个目标，约 \$0.28
osmedeus cloud run \
  -f fast -T targets.txt --instances 20 \
  --sync-back --auto-destroy
```

成本：20 x \$0.007 x 2 小时 = **\$0.28**

### 自定义命令管道

```bash theme={null}
osmedeus cloud run \
  --custom-cmd "subfinder -d {{Target}} -o /tmp/osm-custom/subs.txt" \
  --custom-cmd "cat /tmp/osm-custom/subs.txt | httpx -o /tmp/osm-custom/live.txt" \
  --custom-cmd "cat /tmp/osm-custom/live.txt | nuclei -o /tmp/osm-custom/nuclei.txt" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy
```

成本：\~\$0.007

### 大规模分布式 Nuclei

```bash theme={null}
# 将 10,000 个 URL 分配到 10 个 Hetzner worker
osmedeus cloud run \
  --custom-cmd "cat {{Target}} | nuclei -o /tmp/osm-custom/results.txt" \
  --sync-path "/tmp/osm-custom/results.txt" \
  --sync-dest "./nuclei-hetzner" \
  -T urls.txt --instances 10 --auto-destroy
```

成本：10 x \$0.007 x 1 小时 = **\$0.07**

### 端口扫描

```bash theme={null}
# 使用更大的实例进行 masscan（需要更多资源）
osmedeus cloud config set providers.hetzner.server_type cx32

osmedeus cloud run \
  --custom-cmd "masscan {{Target}} -p1-65535 --rate 10000 -oG /tmp/osm-custom/masscan.txt" \
  --custom-cmd "cat /tmp/osm-custom/masscan.txt | grep 'Host:' | awk '{print \$2\":\" \$5}' | sed 's|/.*||' > /tmp/osm-custom/open-ports.txt" \
  --sync-path "/tmp/osm-custom/" \
  -t 203.0.113.0/24 --auto-destroy
```

### 持久低成本实验室

```bash theme={null}
# 创建 5 个 worker 并保持全天运行
osmedeus cloud create --provider hetzner -n 5

# 全天运行多次扫描
osmedeus cloud run -f fast -t target1.com --reuse
osmedeus cloud run --custom-cmd "nmap -sV {{Target}}" -t target2.com --reuse
osmedeus cloud run -f general -T targets.txt --reuse

# 一天结束时销毁
osmedeus cloud destroy all --force
```

成本：5 x \$0.007 x 8 小时 = **\$0.28**，5 台机器全天扫描。

### 基于欧盟的扫描

当需要从欧盟 IP 空间发起扫描时，Hetzner 的欧洲位置非常有用：

```bash theme={null}
# 使用德国数据中心
osmedeus cloud config set providers.hetzner.location fsn1
osmedeus cloud run -f fast -t eu-target.com --auto-destroy

# 使用芬兰数据中心
osmedeus cloud config set providers.hetzner.location hel1
osmedeus cloud run -f fast -t nordic-target.com --auto-destroy
```

### 使用 cx42 的高性能

```bash theme={null}
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

成本：\~\$0.025 每小时

## Cost Comparison

为什么 Hetzner 是批量扫描最便宜的选择：

| 场景                     | Hetzner (cx22) | DigitalOcean (s-2vcpu-4gb) | AWS (t3.medium) |
| ---------------------- | -------------- | -------------------------- | --------------- |
| 1 实例 x 1 小时    | \$0.007        | \$0.022                    | \$0.042         |
| 5 实例 x 2 小时  | \$0.07         | \$0.22                     | \$0.42          |
| 10 实例 x 4 小时 | \$0.28         | \$0.89                     | \$1.66          |
| 20 实例 x 8 小时 | \$1.12         | \$3.57                     | \$6.66          |

对于相同规格，Hetzner 比 DigitalOcean 便宜约 3 倍，比 AWS 便宜约 6 倍。

## Troubleshooting

### "Unauthorized" 错误

您的 API 令牌无效或已过期。在 Hetzner Cloud Console 中生成一个新令牌。

```bash theme={null}
osmedeus cloud config set providers.hetzner.token <new-token>
```

### 服务器类型不可用

某些服务器类型可能在某些位置不可用。尝试其他位置：

```bash theme={null}
osmedeus cloud config set providers.hetzner.location nbg1
```

### SSH 连接问题

Hetzner 服务器默认使用 `root` 用户：

```bash theme={null}
osmedeus cloud config set ssh.user root
```

验证您的 SSH 密钥配置正确：

```bash theme={null}
osmedeus cloud run --custom-cmd "whoami" -t test --verbose-setup
```

### 速率限制

Hetzner 的 API 有速率限制。如果一次性创建大量实例，可能会触发限制。请错开创建时间或联系 Hetzner 支持提高限制。

### 清理

```bash theme={null}
# 列出所有基础设施
osmedeus cloud list

# 销毁特定实例
osmedeus cloud destroy <infra-id>

# 销毁所有实例
osmedeus cloud destroy all --force

# 如果不同步，直接检查 Hetzner Console：
# Console > Servers > 查找以 osmedeus- 为前缀的服务器
```

## Best Practices

1. **默认使用 cx22** —— 2 vCPU / 4 GB 对于大多数扫描来说足够，价格为 \$0.007/小时
2. **水平扩展** —— 对于可并行化的工作负载，10 个 cx22 比 1 个 cx52 更便宜且更快
3. **始终使用 `--auto-destroy`** —— 即使价格为 \$0.007/小时，遗忘的实例也会累积成本
4. **使用欧洲位置**（fsn1, nbg1）以获得到 Hetzner 网络的最低延迟
5. **预构建快照**用于常用工具配置，以跳过设置时间
6. **设置适度的成本限制** —— 即使 \$5.00 的 max_total_spend 在 Hetzner 定价下也能走很远
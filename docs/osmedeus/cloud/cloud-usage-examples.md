# Cloud 使用示例

在云基础设施上运行分布式安全扫描和自定义命令的实用示例。

## 入门

### 5 分钟内完成首次扫描

```bash
# 步骤 0：启用云功能
osmedeus config set cloud.enabled true

# 步骤 1：配置 AWS 凭据
osmedeus cloud config set providers.aws.access_key_id ${AWS_ACCESS_KEY_ID}
osmedeus cloud config set providers.aws.secret_access_key ${AWS_SECRET_ACCESS_KEY}
osmedeus cloud config set providers.aws.region ap-southeast-1
osmedeus cloud config set defaults.provider aws

# 步骤 2：SSH 密钥
osmedeus cloud config set ssh.private_key_path ~/.ssh/id_rsa
osmedeus cloud config set ssh.public_key_path ~/.ssh/id_rsa.pub

# 步骤 3：先清空设置脚本，然后为工作节点添加设置命令
osmedeus cloud config set setup.commands.clear ""
osmedeus cloud config set setup.commands.add "curl -fsSL https://www.osmedeus.org/install.sh | bash"
osmedeus cloud config set setup.commands.add "osmedeus install base --preset"

# 步骤 4：运行您的首次扫描
osmedeus cloud run -f fast -t example.com --auto-destroy
```

这将预配 1 个 AWS 实例，安装 osmedeus 和工具，对 `example.com` 运行 `fast` 流程，将输出流式传输到您的终端，并在完成后销毁实例。

### 验证配置

```bash
# 查看所有设置
osmedeus cloud config list

# 查看包含秘密的设置
osmedeus cloud config list --show-secrets
```

## 工作流模式示例

### 单目标

```bash
# 运行 flow
osmedeus cloud run -f fast -t example.com

# 运行特定 module
osmedeus cloud run -m enum-subdomain -t example.com

# 带超时
osmedeus cloud run -f general -t example.com --timeout 2h

# 使用特定供应商
osmedeus cloud run -f fast -t example.com --provider digitalocean
```

### 多目标

```bash
# 将目标分发到 5 个工作节点
osmedeus cloud run -f fast -T targets.txt --instances 5

# 每个工作节点 10 个目标（自动计算工作节点数）
osmedeus cloud run -f fast -T targets.txt --instances 10 --chunk-size 10

# 精确分成 3 块
osmedeus cloud run -f fast -T targets.txt --instances 5 --chunk-count 3
```

### 完整生命周期

```bash
# 预配、扫描、同步结果、然后销毁
osmedeus cloud run -f fast -t example.com --sync-back --auto-destroy

# 多目标相同操作
osmedeus cloud run -f fast -T targets.txt --instances 3 --sync-back --auto-destroy
```

### 复用基础设施

```bash
# 首次运行：预配并扫描
osmedeus cloud run -f fast -t target1.com

# 第二次运行：为不同目标复用相同实例
osmedeus cloud run -f fast -t target2.com --reuse

# 按 IP 复用特定机器
osmedeus cloud run -f fast -t target3.com --reuse-with "1.2.3.4,5.6.7.8"

# 完成后手动销毁
osmedeus cloud destroy <infra-id>
```

## 自定义命令示例

### 基本用法

```bash
# 运行单个命令
osmedeus cloud run --custom-cmd "nmap -sV {{Target}}" -t example.com

# 在现有基础设施上运行
osmedeus cloud run --custom-cmd "whoami && id" -t example.com --reuse
```

### 侦察管道

```bash
# 子域名枚举 → HTTP 探测 → 截图
osmedeus cloud run \
  --custom-cmd "subfinder -d {{Target}} -o /tmp/osm-custom/subs.txt" \
  --custom-cmd "cat /tmp/osm-custom/subs.txt | httpx -o /tmp/osm-custom/live.txt" \
  --custom-cmd "cat /tmp/osm-custom/live.txt | gowitness scan -o /tmp/osm-custom/screenshots" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy
```

### 漏洞扫描

```bash
# 使用自定义模板的 Nuclei 扫描
osmedeus cloud run \
  --custom-cmd "nuclei -u {{Target}} -t cves/ -o /tmp/osm-custom/cves.txt" \
  --custom-cmd "nuclei -u {{Target}} -t exposures/ -o /tmp/osm-custom/exposures.txt" \
  --custom-post-cmd "cat /tmp/osm-custom/cves.txt /tmp/osm-custom/exposures.txt | sort -u > /tmp/osm-custom/all-findings.txt" \
  --sync-path "/tmp/osm-custom/all-findings.txt" \
  -t example.com
```

### 大规模端口扫描

```bash
# 将 IP 列表分发到 10 个工作节点进行 masscan + nmap
osmedeus cloud run \
  --custom-cmd "cat {{Target}} | while read ip; do masscan \\$ip -p1-65535 --rate 1000 -oG /tmp/osm-custom/masscan-\\$(echo \\$ip | tr '.' '-').txt; done" \
  --custom-post-cmd "cat /tmp/osm-custom/masscan-*.txt > /tmp/osm-custom/all-ports.txt" \
  --sync-path "/tmp/osm-custom/" \
  -T ip-list.txt --instances 10 --auto-destroy
```

### SAST 扫描

```bash
# 克隆仓库并运行 semgrep
osmedeus cloud run \
  --custom-cmd "git clone https://github.com/org/repo.git /tmp/osm-custom/repo" \
  --custom-cmd "semgrep --config auto /tmp/osm-custom/repo --sarif -o /tmp/osm-custom/semgrep.sarif" \
  --sync-path "/tmp/osm-custom/semgrep.sarif" \
  -t org/repo --auto-destroy
```

### 自定义同步目标

```bash
# 下载到特定本地目录
osmedeus cloud run \
  --custom-cmd "nmap -sV {{Target}} -oA /tmp/osm-custom/nmap" \
  --sync-path "/tmp/osm-custom/" \
  --sync-dest "./nmap-results" \
  -t example.com

# 结果存放于：./nmap-results/<worker-name>-<ip>/tmp/osm-custom/nmap.*
```

### 使用工作节点变量

```bash
# 在扫描结果旁记录工作节点信息
osmedeus cloud run \
  --custom-cmd "echo 'Worker {{worker_name}} ({{public_ip}}) scanning {{Target}}' > /tmp/osm-custom/info.txt" \
  --custom-cmd "nmap -sV {{Target}} -oA /tmp/osm-custom/nmap" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com
```

## 真实场景

### 漏洞赏金：枚举多个项目

```bash
# targets.txt 包含：hackerone.com、bugcrowd.com、intigriti.com...
osmedeus cloud run \
  -f general -T targets.txt --instances 5 \
  --sync-back --auto-destroy --provider digitalocean
```

### 扫描大范围 IP

```bash
# ip-ranges.txt 包含 CIDR 范围，每行一个
osmedeus cloud run \
  --custom-cmd "cat {{Target}} | nmap -iL - -sV -oA /tmp/osm-custom/scan" \
  --sync-path "/tmp/osm-custom/" \
  -T ip-ranges.txt --instances 10 --auto-destroy
```

### 持久活动

```bash
# 一次性创建基础设施
osmedeus cloud create --provider aws -n 3

# 随时间运行多次扫描
osmedeus cloud run -f fast -t target1.com --reuse
osmedeus cloud run -f fast -t target2.com --reuse
osmedeus cloud run --custom-cmd "nuclei -u target3.com -o /tmp/osm-custom/nuclei.txt" \
  --sync-path "/tmp/osm-custom/" -t target3.com --reuse

# 活动结束时销毁
osmedeus cloud destroy all --force
```

### 多供应商策略

```bash
# 使用 Hetzner 进行廉价批量扫描
osmedeus cloud run -f fast -T targets.txt --instances 10 --provider hetzner

# 使用 AWS 处理需要特定地理位置的目标
osmedeus cloud run -f fast -t us-target.com --provider aws
```

## 供应商配置示例

### AWS

```bash
osmedeus cloud config set providers.aws.access_key_id ${AWS_ACCESS_KEY_ID}
osmedeus cloud config set providers.aws.secret_access_key ${AWS_SECRET_ACCESS_KEY}
osmedeus cloud config set providers.aws.region ap-southeast-1
osmedeus cloud config set providers.aws.instance_type t3.medium
osmedeus cloud config set providers.aws.use_spot true
osmedeus cloud config set defaults.provider aws
```

详见 [AWS 供应商指南](./cloud-provider-aws.md) 了解详细设置和示例。

### Hetzner

```bash
osmedeus cloud config set providers.hetzner.token ${HETZNER_API_TOKEN}
osmedeus cloud config set providers.hetzner.location fsn1
osmedeus cloud config set providers.hetzner.server_type cx22
osmedeus cloud config set defaults.provider hetzner
```

详见 [Hetzner 供应商指南](./cloud-provider-hetzner.md) 了解详细设置和示例。

### DigitalOcean

```bash
osmedeus cloud config set providers.digitalocean.token ${DO_TOKEN}
osmedeus cloud config set providers.digitalocean.region sgp1
osmedeus cloud config set providers.digitalocean.size s-2vcpu-4gb
osmedeus cloud config set defaults.provider digitalocean
```

### GCP

```bash
osmedeus cloud config set providers.gcp.project_id ${GCP_PROJECT}
osmedeus cloud config set providers.gcp.credentials_file /path/to/sa-key.json
osmedeus cloud config set providers.gcp.region us-central1
osmedeus cloud config set providers.gcp.zone us-central1-a
osmedeus cloud config set providers.gcp.machine_type n1-standard-2
osmedeus cloud config set providers.gcp.use_preemptible true
osmedeus cloud config set defaults.provider gcp
```

### Linode

```bash
osmedeus cloud config set providers.linode.token ${LINODE_TOKEN}
osmedeus cloud config set providers.linode.region ap-south
osmedeus cloud config set providers.linode.type g6-standard-2
osmedeus cloud config set defaults.provider linode
```

### Azure

```bash
osmedeus cloud config set providers.azure.subscription_id ${AZURE_SUB_ID}
osmedeus cloud config set providers.azure.tenant_id ${AZURE_TENANT_ID}
osmedeus cloud config set providers.azure.client_id ${AZURE_CLIENT_ID}
osmedeus cloud config set providers.azure.client_secret ${AZURE_CLIENT_SECRET}
osmedeus cloud config set providers.azure.location southeastasia
osmedeus cloud config set providers.azure.vm_size Standard_B2s
osmedeus cloud config set defaults.provider azure
```

## 高级主题

### 自定义快照

在 VM 上预安装工具，创建快照，然后使用该快照以加快启动速度：

```bash
# 1. 通过供应商控制台手动创建和设置 VM
# 2. 安装 osmedeus + 所有工具
# 3. 在供应商控制台中创建快照/镜像
# 4. 配置 osmedeus 使用它

# AWS
osmedeus cloud config set providers.aws.ami ami-0123456789abcdef0

# DigitalOcean
osmedeus cloud config set providers.digitalocean.snapshot_id 12345678

# Hetzner
osmedeus cloud config set providers.hetzner.image 12345678
```

启动时间从约 5 分钟降至约 30 秒。

### 自定义工作节点设置

```bash
# 添加设置命令（在每个工作节点上按顺序运行）
osmedeus cloud config set setup.commands.add "apt-get update && apt-get install -y nmap masscan"
osmedeus cloud config set setup.commands.add "go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest"

# 添加后置设置命令（带每个工作节点的变量展开）
osmedeus cloud config set setup.post_commands.add "echo '{{worker_name}} at {{public_ip}}' >> /tmp/workers.txt"
```

### Ansible 设置

```bash
osmedeus cloud config set setup.ansible.enabled true
osmedeus cloud config set setup.ansible.playbook_path /path/to/setup.yaml
osmedeus cloud run -f fast -t example.com --ansible
```

### 环境变量展开

所有配置值支持 `${ENV_VAR}` 语法：

```bash
osmedeus cloud config set providers.aws.access_key_id '${AWS_ACCESS_KEY_ID}'
osmedeus cloud config set providers.aws.secret_access_key '${AWS_SECRET_ACCESS_KEY}'
```

值在运行时从您的 Shell 环境中展开。

## 故障排除

### 工作节点无法连接
```bash
osmedeus cloud run -f fast -t example.com --verbose-setup  # 查看 SSH 输出
osmedeus cloud run -f fast -t example.com --debug          # 完整调试日志
```

### 孤儿基础设施
```bash
osmedeus cloud list                    # 检查正在运行的内容
osmedeus cloud destroy all --force     # 紧急清理
```

### 超出成本限制
```bash
osmedeus cloud config set limits.max_hourly_spend 5.00   # 增加限制
```

### 自定义命令失败
- 检查工具是否已在您的设置命令中安装
- 使用 `--verbose-setup` 验证设置是否完成
- 先用简单命令测试：`--custom-cmd "which nmap"`

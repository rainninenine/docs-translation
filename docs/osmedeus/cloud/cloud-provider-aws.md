# AWS 供应商指南

在 AWS EC2 实例上运行 osmedeus cloud 的分步指南。

## 前提条件

- 一个 AWS 账户
- 一个具有 EC2 权限的 IAM 用户或角色
- 一个 SSH 密钥对（本地 `~/.ssh/id_rsa` 和 `~/.ssh/id_rsa.pub`）

### 所需 IAM 权限

IAM 用户需要以下权限（或使用 `AmazonEC2FullAccess` 托管策略）：

```
ec2:RunInstances
ec2:TerminateInstances
ec2:DescribeInstances
ec2:DescribeImages
ec2:CreateSecurityGroup
ec2:AuthorizeSecurityGroupIngress
ec2:DeleteSecurityGroup
ec2:DescribeSecurityGroups
ec2:ImportKeyPair
ec2:DeleteKeyPair
ec2:DescribeKeyPairs
ec2:CreateTags
```

### 获取凭据

1. 前往 **IAM 控制台** > **用户** > 选择您的用户
2. **安全凭据** 标签 > **创建访问密钥**
3. 保存 **访问密钥 ID** 和 **秘密访问密钥**

或者，如果已为 AWS CLI 配置了环境变量：

```bash
export AWS_ACCESS_KEY_ID=<YOUR_AWS_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<YOUR_AWS_SECRET_ACCESS_KEY>
```

## 配置

### 最小设置

```bash
# 启用云功能
osmedeus config set cloud.enabled true

# 凭据
osmedeus cloud config set providers.aws.access_key_id ${AWS_ACCESS_KEY_ID}
osmedeus cloud config set providers.aws.secret_access_key ${AWS_SECRET_ACCESS_KEY}
osmedeus cloud config set providers.aws.region ap-southeast-1
osmedeus cloud config set defaults.provider aws

# SSH
osmedeus cloud config set ssh.private_key_path ~/.ssh/id_rsa
osmedeus cloud config set ssh.public_key_path ~/.ssh/id_rsa.pub
osmedeus cloud config set ssh.user ubuntu

# 先清空设置脚本
osmedeus cloud config set setup.commands.clear ""

# 工作节点设置
osmedeus cloud config set setup.commands.add "sudo apt-get update"
osmedeus cloud config set setup.commands.add "sudo apt-get install -y -qq curl git tmux unzip jq rsync"
osmedeus cloud config set setup.commands.add "curl -fsSL https://www.osmedeus.org/install.sh | bash"
osmedeus cloud config set setup.commands.add "osmedeus health"
```

### 实例类型

| 实例 | vCPU | RAM | $/时（按需） | $/时（竞价，约省 70%） | 最佳用途 |
|----------|------|-----|------|------|----------|
| t3.medium | 2 | 4 GB | $0.0416 | ~$0.012 | 轻量扫描，单目标 |
| t3.large | 2 | 8 GB | $0.0832 | ~$0.025 | 通用扫描 |
| t3.xlarge | 4 | 16 GB | $0.1664 | ~$0.050 | 重型扫描，大型目标列表 |
| t3.2xlarge | 8 | 32 GB | $0.3328 | ~$0.100 | 并行管道 |

```bash
# 设置实例类型
osmedeus cloud config set providers.aws.instance_type t3.large
```

### 竞价实例

竞价实例比按需实例便宜 60-80%。它们可能被中断，但适合安全扫描（无状态，可重试）。

```bash
osmedeus cloud config set providers.aws.use_spot true
```

### 区域

选择靠近目标或价格最低的区域：

| 区域 | 位置 | 代码 |
|--------|----------|------|
| 美国东部（弗吉尼亚北部） | 美国 | `us-east-1` |
| 美国西部（俄勒冈） | 美国 | `us-west-2` |
| 欧洲（法兰克福） | 欧洲 | `eu-central-1` |
| 欧洲（爱尔兰） | 欧洲 | `eu-west-1` |
| 亚太（新加坡） | 亚洲 | `ap-southeast-1` |
| 亚太（东京） | 亚洲 | `ap-northeast-1` |
| 亚太（孟买） | 亚洲 | `ap-south-1` |
| 亚太（悉尼） | 澳大利亚 | `ap-southeast-2` |

```bash
osmedeus cloud config set providers.aws.region us-east-1
```

### 自定义 AMI

使用预装工具的自定义 AMI 以加快启动速度：

```bash
# 查找您所在区域的默认 Ubuntu AMI
# aws ec2 describe-images --owners 099720109477 --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-*-amd64-*" --query 'sort_by(Images, &CreationDate)[-1].ImageId'

# 或使用您自己预构建的 AMI
osmedeus cloud config set providers.aws.ami ami-0123456789abcdef0
```

### 成本限制

```bash
osmedeus cloud config set limits.max_hourly_spend 1.00
osmedeus cloud config set limits.max_total_spend 10.00
osmedeus cloud config set limits.max_instances 10
```

## 示例

### 快速域名侦察

```bash
osmedeus cloud run -f fast -t example.com --auto-destroy
```

成本：~$0.04（1 x t3.medium x 1 小时）

### 大规模子域名枚举

```bash
# targets.txt：每行一个域名
osmedeus cloud run -f general -T targets.txt --instances 5 --sync-back --auto-destroy
```

成本：~$0.42（5 x t3.medium x 2 小时）

### 自定义 Nmap 扫描

```bash
osmedeus cloud run \
  --custom-cmd "nmap -sV -sC {{Target}} -oA /tmp/osm-custom/nmap" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy
```

### 分布式 Nuclei 扫描

```bash
osmedeus cloud run \
  --custom-cmd "cat {{Target}} | nuclei -o /tmp/osm-custom/results.txt" \
  --sync-path "/tmp/osm-custom/results.txt" \
  --sync-dest "./nuclei-aws" \
  -T urls.txt --instances 10 --auto-destroy
```

成本：~$0.42（10 x t3.medium x 1 小时）

### 竞价实例管道

```bash
# 配置竞价
osmedeus cloud config set providers.aws.use_spot true
osmedeus cloud config set providers.aws.instance_type t3.large

# 低成本运行重型扫描
osmedeus cloud run \
  --custom-cmd "subfinder -d {{Target}} -all -o /tmp/osm-custom/subs.txt" \
  --custom-cmd "cat /tmp/osm-custom/subs.txt | httpx -td -o /tmp/osm-custom/live.txt" \
  --custom-cmd "cat /tmp/osm-custom/live.txt | nuclei -o /tmp/osm-custom/nuclei.txt" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy
```

成本：~$0.025（1 x t3.large 竞价 x 1 小时）

### 持久侦察活动

```bash
# 一次性创建实例（节省后续运行的设置时间）
osmedeus cloud create --provider aws -n 3

# 全天运行扫描
osmedeus cloud run -f fast -t target1.com --reuse
osmedeus cloud run -f fast -t target2.com --reuse
osmedeus cloud run --custom-cmd "nuclei -u target3.com -o /tmp/osm-custom/nuclei.txt" \
  --sync-path "/tmp/osm-custom/" -t target3.com --reuse

# 一天结束时销毁
osmedeus cloud destroy all --force
```

### 多区域扫描

```bash
# 从美国区域扫描美国目标
osmedeus cloud config set providers.aws.region us-east-1
osmedeus cloud run -f fast -t us-company.com --auto-destroy

# 从新加坡扫描亚太目标
osmedeus cloud config set providers.aws.region ap-southeast-1
osmedeus cloud run -f fast -t apac-company.com --auto-destroy
```

## 故障排除

### "UnauthorizedOperation" 错误

您的 IAM 用户缺少所需权限。请附加 `AmazonEC2FullAccess` 策略或上面列出的最小权限。

### 实例无法启动

```bash
# 使用调试输出检查
osmedeus cloud run -f fast -t example.com --debug

# 常见原因：
# - 区域中没有可用的实例类型
# - 已达到 vCPU 限制（在 AWS 控制台中请求增加限制）
# - 竞价容量不可用（尝试按需或不同区域）
```

### SSH 连接超时

```bash
# 验证安全组允许 SSH（端口 22）
# 使用详细设置检查
osmedeus cloud run -f fast -t example.com --verbose-setup
```

### 竞价实例被中断

竞价实例可能被 AWS 回收。该工作节点的扫描将失败。缓解措施：

- 使用 `--auto-destroy` 进行清理
- 重新运行失败的目标
- 对关键扫描使用按需实例

### 清理

```bash
# 列出所有基础设施
osmedeus cloud list

# 销毁特定资源
osmedeus cloud destroy <infra-id>

# 核弹选项
osmedeus cloud destroy all --force

# 如果 osmedeus 状态不同步，直接检查 AWS 控制台：
# EC2 控制台 > 实例 > 按标签 "osmedeus" 过滤
```

## 成本优化

1. **对所有非关键扫描使用竞价实例**（`use_spot: true`）
2. **合理选择实例大小**：t3.medium 足以满足大多数单目标扫描
3. **始终使用 `--auto-destroy`** 防止遗忘实例
4. **设置成本限制**以防止失控支出
5. **使用自定义 AMI** 减少设置时间（减少实例小时数）
6. **选择最便宜的区域**（如果目标地理位置不重要，us-east-1 通常最便宜）

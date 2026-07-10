> ## Documentation Index
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# GCP Provider Guide

> 在 Google Cloud Platform Compute Engine 实例上运行 osmedeus cloud 的分步指南

在 Google Cloud Platform Compute Engine 实例上运行 osmedeus cloud 的分步指南。

## Prerequisites

* 一个包含项目的 GCP 账户
* 一个具有 Compute Engine 权限的服务账号
* 一个服务账号密钥文件（JSON）
* 一个 SSH 密钥对（本地 `~/.ssh/id_rsa` 和 `~/.ssh/id_rsa.pub`）

### Required IAM Permissions

服务账号需要以下角色（或使用 `Compute Admin` 角色）：

```
compute.instances.create
compute.instances.delete
compute.instances.get
compute.instances.list
compute.instances.setMetadata
compute.firewalls.create
compute.firewalls.delete
compute.firewalls.get
compute.networks.get
compute.subnetworks.use
compute.disks.create
compute.images.get
compute.images.useReadOnly
```

最简单的方法是为服务账号分配 **Compute Admin**（`roles/compute.admin`）角色。

### Create a Service Account and Key

1. 进入 **IAM & Admin** > **Service Accounts** > **Create Service Account**
2. 命名为 `osmedeus-cloud`（或类似名称）
3. 授予 **Compute Admin** 角色
4. 进入服务账号 > **Keys** > **Add Key** > **Create new key** > **JSON**
5. 保存 JSON 文件（例如 `~/.gcp/osmedeus-sa.json`）

或者通过 `gcloud` CLI：

```bash theme={null}
# 创建服务账号
gcloud iam service-accounts create osmedeus-cloud \
  --display-name="Osmedeus Cloud Scanner"

# 授予 Compute Admin 角色
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:osmedeus-cloud@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/compute.admin"

# 创建并下载密钥文件
gcloud iam service-accounts keys create ~/.gcp/osmedeus-sa.json \
  --iam-account=osmedeus-cloud@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

你也可以将凭证文件路径导出为环境变量：

```bash theme={null}
export GCP_PROJECT_ID=your-project-id
export GCP_CREDENTIALS_FILE=~/.gcp/osmedeus-sa.json
```

## Configuration

### Minimal Setup

```bash theme={null}
# 启用云功能
osmedeus config set cloud.enabled true

# 凭证
osmedeus cloud config set providers.gcp.project_id ${GCP_PROJECT_ID}
osmedeus cloud config set providers.gcp.credentials_file ${GCP_CREDENTIALS_FILE}
osmedeus cloud config set providers.gcp.region us-central1
osmedeus cloud config set providers.gcp.zone us-central1-a
osmedeus cloud config set defaults.provider gcp

# SSH
osmedeus cloud config set ssh.private_key_path ~/.ssh/id_rsa
osmedeus cloud config set ssh.public_key_path ~/.ssh/id_rsa.pub
osmedeus cloud config set ssh.user root

# 先清空设置脚本
osmedeus cloud config set setup.commands.clear ""

# Worker 设置
osmedeus cloud config set setup.commands.add "sudo apt-get update"
osmedeus cloud config set setup.commands.add "sudo apt-get install -y -qq curl git tmux unzip jq rsync"
osmedeus cloud config set setup.commands.add "curl -fsSL https://www.osmedeus.org/install.sh | bash"
osmedeus cloud config set setup.commands.add "osmedeus install base --preset"
```

### Machine Types

| Machine Type  | vCPU | RAM    | \$/hr (按需) | \$/hr (抢占式，约省80%) | 最佳用途                        |
| ------------- | ---- | ------ | ----------------- | ------------------------------ | ------------------------------- |
| e2-medium     | 2    | 4 GB   | \$0.0335          | \~\$0.010                      | 轻量扫描，单个目标     |
| n1-standard-2 | 2    | 7.5 GB | \$0.0950          | \~\$0.019                      | 通用扫描（默认）      |
| n1-standard-4 | 4    | 15 GB  | \$0.1900          | \~\$0.038                      | 重型扫描，大型目标列表 |
| n2-standard-2 | 2    | 8 GB   | \$0.0971          | \~\$0.019                      | 通用扫描（新一代）    |
| n2-standard-4 | 4    | 16 GB  | \$0.1942          | \~\$0.039                      | 并行管道              |
| c2-standard-4 | 4    | 16 GB  | \$0.2088          | \~\$0.042                      | CPU 密集型扫描             |

```bash theme={null}
# 设置机器类型
osmedeus cloud config set providers.gcp.machine_type n1-standard-2
```

### Preemptible Instances

抢占式虚拟机成本比按需实例低 80%。它们最长运行 24 小时，可能被回收，但非常适合安全扫描工作负载。

```bash theme={null}
osmedeus cloud config set providers.gcp.use_preemptible true
```

### Regions and Zones

选择靠近目标或定价最低的区域：

| Region         | 位置  | 代码                   | 可用区示例             |
| -------------- | --------- | ---------------------- | ------------------------ |
| Iowa           | 美国        | `us-central1`          | `us-central1-a`          |
| South Carolina | 美国        | `us-east1`             | `us-east1-b`             |
| Oregon         | 美国        | `us-west1`             | `us-west1-b`             |
| Frankfurt      | 欧洲    | `europe-west3`         | `europe-west3-a`         |
| London         | 欧洲    | `europe-west2`         | `europe-west2-a`         |
| Singapore      | 亚洲      | `asia-southeast1`      | `asia-southeast1-a`      |
| Tokyo          | 亚洲      | `asia-northeast1`      | `asia-northeast1-a`      |
| Mumbai         | 亚洲      | `asia-south1`          | `asia-south1-a`          |
| Sydney         | 澳大利亚 | `australia-southeast1` | `australia-southeast1-a` |

```bash theme={null}
osmedeus cloud config set providers.gcp.region us-central1
osmedeus cloud config set providers.gcp.zone us-central1-a
```

> **注意：** 可用区必须在所选区域内。

### Custom Image Family

使用预装工具的自定义镜像系列可加快启动速度：

```bash theme={null}
# 默认使用 ubuntu-os-cloud 项目中的 ubuntu-2204-lts
# 如果你有自己的自定义镜像系列，请使用它
osmedeus cloud config set providers.gcp.image_family my-osmedeus-image
```

### Cost Limits

```bash theme={null}
osmedeus cloud config set limits.max_hourly_spend 1.00
osmedeus cloud config set limits.max_total_spend 10.00
osmedeus cloud config set limits.max_instances 10
```

## Examples

### Quick Domain Recon

```bash theme={null}
osmedeus cloud run -f fast -t example.com --auto-destroy
```

成本：约 \$0.03（1 个 e2-medium x 1 小时）

### Large-Scale Subdomain Enumeration

```bash theme={null}
# targets.txt：每行一个域名
osmedeus cloud run -f general -T targets.txt --instances 5 --sync-back --auto-destroy
```

成本：约 \$0.48（5 个 n1-standard-2 x 1 小时）

### Custom Nmap Scan

```bash theme={null}
osmedeus cloud run \
  --custom-cmd "nmap -sV -sC {{Target}} -oA /tmp/osm-custom/nmap" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy
```

### Distributed Nuclei Scanning

```bash theme={null}
osmedeus cloud run \
  --custom-cmd "cat {{Target}} | nuclei -o /tmp/osm-custom/results.txt" \
  --sync-path "/tmp/osm-custom/results.txt" \
  --sync-dest "./nuclei-gcp" \
  -T urls.txt --instances 10 --auto-destroy
```

成本：约 \$0.34（10 个 e2-medium x 1 小时）

### Preemptible Instance Pipeline

```bash theme={null}
# 配置抢占式
osmedeus cloud config set providers.gcp.use_preemptible true
osmedeus cloud config set providers.gcp.machine_type n1-standard-2

# 低成本运行重型扫描
osmedeus cloud run \
  --custom-cmd "subfinder -d {{Target}} -all -o /tmp/osm-custom/subs.txt" \
  --custom-cmd "cat /tmp/osm-custom/subs.txt | httpx -td -o /tmp/osm-custom/live.txt" \
  --custom-cmd "cat /tmp/osm-custom/live.txt | nuclei -o /tmp/osm-custom/nuclei.txt" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy
```

成本：约 \$0.019（1 个 n1-standard-2 抢占式 x 1 小时）

### Persistent Recon Campaign

```bash theme={null}
# 一次性创建实例（后续运行节省设置时间）
osmedeus cloud create --provider gcp -n 3

# 全天运行扫描
osmedeus cloud run -f fast -t target1.com --reuse
osmedeus cloud run -f fast -t target2.com --reuse
osmedeus cloud run --custom-cmd "nuclei -u target3.com -o /tmp/osm-custom/nuclei.txt" \
  --sync-path "/tmp/osm-custom/" -t target3.com --reuse

# 一天结束后销毁
osmedeus cloud destroy all --force
```

### Multi-Region Scanning

```bash theme={null}
# 从爱荷华州扫描美国目标
osmedeus cloud config set providers.gcp.region us-central1
osmedeus cloud config set providers.gcp.zone us-central1-a
osmedeus cloud run -f fast -t us-company.com --auto-destroy

# 从新加坡扫描亚太目标
osmedeus cloud config set providers.gcp.region asia-southeast1
osmedeus cloud config set providers.gcp.zone asia-southeast1-a
osmedeus cloud run -f fast -t apac-company.com --auto-destroy
```

## Troubleshooting

### "Permission denied" 或 "403 Forbidden"

服务账号缺少所需权限。分配 **Compute Admin** 角色：

```bash theme={null}
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:YOUR_SA@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/compute.admin"
```

### "Credentials file not found"

确保 JSON 密钥文件路径正确且文件存在：

```bash theme={null}
# 检查文件是否存在
ls -la ~/.gcp/osmedeus-sa.json

# 或通过环境变量设置
export GCP_CREDENTIALS_FILE=/absolute/path/to/key.json
osmedeus cloud config set providers.gcp.credentials_file ${GCP_CREDENTIALS_FILE}
```

### Instances Not Starting

```bash theme={null}
# 使用调试输出检查
osmedeus cloud run -f fast -t example.com --debug

# 常见原因：
# - 配额超限（在 Cloud Console 中检查 Quotas 页面）
# - 可用区没有可用的机器类型
# - Compute Engine API 未启用（在 APIs & Services 中启用）
# - 抢占式容量不可用（尝试其他可用区或按需实例）
```

### "Compute Engine API has not been used" Error

为项目启用 Compute Engine API：

```bash theme={null}
gcloud services enable compute.googleapis.com --project=YOUR_PROJECT_ID
```

### SSH Connection Timeout

```bash theme={null}
# 验证防火墙规则允许 SSH（端口 22）
gcloud compute firewall-rules list --filter="name~osmedeus"

# 使用详细设置检查
osmedeus cloud run -f fast -t example.com --verbose-setup
```

### Preemptible Instance Terminated

抢占式虚拟机在 24 小时后或 GCP 需要容量时会被回收。该 worker 的扫描将失败。缓解措施：

* 使用 `--auto-destroy` 进行清理
* 重新运行失败的目标
* 对关键或长时间运行的扫描使用按需实例

### Cleaning Up

```bash theme={null}
# 列出所有基础设施
osmedeus cloud list

# 销毁特定资源
osmedeus cloud destroy <infra-id>

# 核选项
osmedeus cloud destroy all --force

# 如果 osmedeus 状态不同步，直接检查 GCP 控制台：
# Compute Engine > VM Instances > 按标签 "osmedeus" 过滤
# 或通过 gcloud：
gcloud compute instances list --filter="labels.osmedeus:*"
```

## Cost Optimization

1. **对所有非关键扫描使用抢占式实例**（`use_preemptible: true`）——最多节省 80%
2. **合理选择机器规格**：e2-medium 对于大多数单目标扫描已足够
3. **始终使用 `--auto-destroy`** 以防止忘记销毁实例
4. **设置成本限制** 以控制失控支出
5. **使用自定义镜像** 减少设置时间（减少实例小时数）
6. **选择最便宜的区域** 如果目标地理位置不重要（us-central1 通常最便宜）
7. **GCP 持续使用折扣** 自动适用于每月运行超过 25% 的按需虚拟机
> ## Documentation Index
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索之前，请使用此文件发现所有可用页面。

# Cloud Cheatsheet

> 云设置、工作流模式、自定义命令和基础设施管理的快速速查表

## First-Time Setup

```bash theme={null}
# 0. 启用云功能
osmedeus config set cloud.enabled true

# 1. 凭证（选择一个供应商）
osmedeus cloud config set providers.aws.access_key_id <key>
osmedeus cloud config set providers.aws.secret_access_key <secret>
osmedeus cloud config set providers.aws.region ap-southeast-1
osmedeus cloud config set defaults.provider aws

# 2. SSH
osmedeus cloud config set ssh.private_key_path ~/.ssh/id_rsa
osmedeus cloud config set ssh.public_key_path ~/.ssh/id_rsa.pub

# 3. 先清理设置脚本，然后添加工作节点设置
osmedeus cloud config set setup.commands.clear ""
osmedeus cloud config set setup.commands.add "curl -fsSL https://www.osmedeus.org/install.sh | bash"
osmedeus cloud config set setup.commands.add "osmedeus install base --preset"

# 4. 成本限制（推荐）
osmedeus cloud config set limits.max_hourly_spend 1.00
osmedeus cloud config set limits.max_total_spend 10.00
```

## Workflow Mode

```bash theme={null}
osmedeus cloud run -f fast -t example.com                          # 单个目标
osmedeus cloud run -f fast -T targets.txt --instances 5            # 分布式
osmedeus cloud run -f fast -t example.com --sync-back              # 同步结果
osmedeus cloud run -f fast -t example.com --auto-destroy           # 自动清理
osmedeus cloud run -f fast -t example.com --sync-back --auto-destroy  # 完整生命周期
osmedeus cloud run -f fast -t example.com --reuse                  # 重用基础设施
osmedeus cloud run -m enum-subdomain -t example.com --timeout 30m  # 模块 + 超时
```

## Custom Command Mode

```bash theme={null}
# 在云实例上运行任意命令
osmedeus cloud run --custom-cmd "nmap -sV {{Target}}" -t example.com

# 流水线：多个命令，同步结果
osmedeus cloud run \
  --custom-cmd "subfinder -d {{Target}} -o /tmp/osm-custom/subs.txt" \
  --custom-cmd "cat /tmp/osm-custom/subs.txt | httpx -o /tmp/osm-custom/live.txt" \
  --custom-post-cmd "wc -l /tmp/osm-custom/live.txt" \
  --sync-path "/tmp/osm-custom/" \
  -t example.com --auto-destroy

# 分发目标，同步到自定义目录
osmedeus cloud run \
  --custom-cmd "cat {{Target}} | nuclei -o /tmp/osm-custom/nuclei.txt" \
  --sync-path "/tmp/osm-custom/nuclei.txt" \
  --sync-dest "./nuclei-results" \
  -T targets.txt --instances 5
```

### 变量：`{{Target}}` `{{public_ip}}` `{{private_ip}}` `{{worker_name}}` `{{worker_id}}` `{{infra_id}}` `{{provider}}` `{{ssh_user}}` `{{index}}`

### 规则

* 命令在远程的 `/tmp/osm-custom/` 目录下运行
* 每个工作节点顺序执行，工作节点之间并行执行
* 第一个失败的命令会跳过剩余的命令和后置命令
* 同步目标：`<sync-dest>/<worker_name>-<ip>/<path>`

## Infrastructure

```bash theme={null}
osmedeus cloud create --provider aws -n 3     # 创建
osmedeus cloud list                           # 列出
osmedeus cloud destroy <id>                   # 销毁单个
osmedeus cloud destroy all --force            # 销毁全部
osmedeus cloud setup --reuse-with "1.2.3.4"   # 设置现有实例
```

## Config

```bash theme={null}
osmedeus cloud config list                    # 查看
osmedeus cloud config set <key> <value>       # 设置
osmedeus cloud config set <key>.add <value>   # 追加到列表
osmedeus cloud config clean                   # 重置
```

## Provider Quick Config

**AWS:**

```bash theme={null}
osmedeus cloud config set providers.aws.access_key_id ${AWS_ACCESS_KEY_ID}
osmedeus cloud config set providers.aws.secret_access_key ${AWS_SECRET_ACCESS_KEY}
osmedeus cloud config set providers.aws.region ap-southeast-1
osmedeus cloud config set providers.aws.instance_type t3.medium
osmedeus cloud config set providers.aws.use_spot true          # 便宜70%
```

**Hetzner:**

```bash theme={null}
osmedeus cloud config set providers.hetzner.token ${HETZNER_API_TOKEN}
osmedeus cloud config set providers.hetzner.location fsn1
osmedeus cloud config set providers.hetzner.server_type cx22
```

**DigitalOcean:**

```bash theme={null}
osmedeus cloud config set providers.digitalocean.token ${DO_TOKEN}
osmedeus cloud config set providers.digitalocean.region sgp1
osmedeus cloud config set providers.digitalocean.size s-2vcpu-4gb
```

**GCP:**

```bash theme={null}
osmedeus cloud config set providers.gcp.project_id ${GCP_PROJECT}
osmedeus cloud config set providers.gcp.credentials_file /path/to/sa-key.json
osmedeus cloud config set providers.gcp.region us-central1
osmedeus cloud config set providers.gcp.zone us-central1-a
osmedeus cloud config set providers.gcp.machine_type n1-standard-2
```

**Linode:**

```bash theme={null}
osmedeus cloud config set providers.linode.token ${LINODE_TOKEN}
osmedeus cloud config set providers.linode.region ap-south
osmedeus cloud config set providers.linode.type g6-standard-2
```

**Azure:**

```bash theme={null}
osmedeus cloud config set providers.azure.subscription_id ${AZURE_SUB_ID}
osmedeus cloud config set providers.azure.tenant_id ${AZURE_TENANT_ID}
osmedeus cloud config set providers.azure.client_id ${AZURE_CLIENT_ID}
osmedeus cloud config set providers.azure.client_secret ${AZURE_CLIENT_SECRET}
osmedeus cloud config set providers.azure.location southeastasia
osmedeus cloud config set providers.azure.vm_size Standard_B2s
```

## Cost Reference

| 供应商       | 实例            | vCPU | 内存    | 美元/小时 |
| ------------ | --------------- | ---- | ------- | --------- |
| Hetzner      | cx22            | 2    | 4 GB    | 0.007     |
| Linode       | g6-standard-2   | 2    | 4 GB    | 0.018     |
| DigitalOcean | s-2vcpu-4gb     | 2    | 4 GB    | 0.022     |
| AWS          | t3.medium       | 2    | 4 GB    | 0.042     |
| Azure        | Standard\_B2s   | 2    | 4 GB    | 0.042     |
| GCP          | n1-standard-2   | 2    | 7.5 GB  | 0.095     |

5 x Hetzner cx22 x 2 小时 = **$0.07** | 5 x DO s-2vcpu-4gb x 2 小时 = **$0.22**

## Troubleshooting

```bash theme={null}
osmedeus cloud run -f fast -t example.com --verbose-setup   # 查看设置输出
osmedeus cloud run -f fast -t example.com --debug            # 完整调试日志
osmedeus cloud list                                          # 检查孤儿实例
osmedeus cloud destroy all --force                           # 紧急清理
```
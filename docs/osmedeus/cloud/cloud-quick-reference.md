# Cloud 快速参考

## 设置（30 秒）

```bash
# 启用云功能
osmedeus config set cloud.enabled true

# 设置凭据（选择您的供应商）
osmedeus cloud config set providers.aws.access_key_id ${AWS_ACCESS_KEY_ID}
osmedeus cloud config set providers.aws.secret_access_key ${AWS_SECRET_ACCESS_KEY}
osmedeus cloud config set providers.aws.region ap-southeast-1
osmedeus cloud config set defaults.provider aws

# SSH 密钥
osmedeus cloud config set ssh.private_key_path ~/.ssh/id_rsa
osmedeus cloud config set ssh.public_key_path ~/.ssh/id_rsa.pub

# 先清空设置脚本
osmedeus cloud config set setup.commands.clear ""

# 工作节点设置命令
osmedeus cloud config set setup.commands.add "curl -fsSL https://www.osmedeus.org/install.sh | bash"
osmedeus cloud config set setup.commands.add "osmedeus install base --preset"
```

## 配置

```bash
osmedeus cloud config list                              # 查看所有设置
osmedeus cloud config set <key> <value>                 # 设置值
osmedeus cloud config set <key>.add <value>             # 追加到列表
osmedeus cloud config clean                             # 重置为默认值

# 供应商凭据
osmedeus cloud config set providers.<provider>.<key> <value>

# 实例类型
osmedeus cloud config set providers.aws.instance_type t3.large

# 竞价实例（便宜 70-80%）
osmedeus cloud config set providers.aws.use_spot true

# 成本限制
osmedeus cloud config set limits.max_hourly_spend 1.00
osmedeus cloud config set limits.max_total_spend 10.00
osmedeus cloud config set limits.max_instances 10
```

## 基础设施

```bash
osmedeus cloud create --provider aws -n 3               # 创建实例
osmedeus cloud list                                     # 列出活跃基础设施
osmedeus cloud destroy <infra-id>                       # 按 ID 销毁
osmedeus cloud destroy all --force                      # 销毁所有
osmedeus cloud setup --reuse-with "1.2.3.4,5.6.7.8"    # 设置现有机器
```

## 工作流模式

```bash
# 基本
osmedeus cloud run -f fast -t example.com
osmedeus cloud run -m enum-subdomain -t example.com --timeout 30m

# 多实例
osmedeus cloud run -f general -t example.com --instances 3 --provider aws

# 跨工作节点分发多个目标
osmedeus cloud run -f fast -T targets.txt --instances 5
osmedeus cloud run -f fast -T targets.txt --chunk-size 10    # 每个工作节点 10 个目标
osmedeus cloud run -f fast -T targets.txt --chunk-count 3    # 分成 3 块

# 复用现有基础设施
osmedeus cloud run -f fast -t example.com --reuse
osmedeus cloud run -f fast -t example.com --reuse-with "1.2.3.4,5.6.7.8"

# 同步结果 + 自动销毁
osmedeus cloud run -f fast -t example.com --sync-back --auto-destroy
```

## 自定义命令模式

在云实例上运行任意命令（与 `-f`/`-m` 互斥）：

```bash
# 单个命令
osmedeus cloud run --custom-cmd "nmap -sV {{Target}}" -t example.com

# 多个顺序命令
osmedeus cloud run \
  --custom-cmd "subfinder -d {{Target}} -o /tmp/osm-custom/subs.txt" \
  --custom-cmd "cat /tmp/osm-custom/subs.txt | httpx -o /tmp/osm-custom/live.txt" \
  -t example.com

# 后置命令（仅在所有自定义命令成功后运行）
osmedeus cloud run \
  --custom-cmd "nuclei -u {{Target}} -o /tmp/osm-custom/results.txt" \
  --custom-post-cmd "cat /tmp/osm-custom/results.txt | notify" \
  -t example.com

# 同步结果
osmedeus cloud run \
  --custom-cmd "nmap -sV {{Target}} -oA /tmp/osm-custom/scan" \
  --sync-path "/tmp/osm-custom/" \
  --sync-dest "./my-results" \
  -t example.com

# 跨工作节点分发目标
osmedeus cloud run \
  --custom-cmd "cat {{Target}} | httpx -o /tmp/osm-custom/live.txt" \
  --sync-path "/tmp/osm-custom/live.txt" \
  -T targets.txt --instances 5 --auto-destroy
```

### 模板变量

| 变量 | 描述 |
|----------|-------------|
| `{{Target}}` | 目标字符串或块文件路径（使用 `-T` 时） |
| `{{public_ip}}` | 工作节点的公网 IP |
| `{{private_ip}}` | 工作节点的私网 IP |
| `{{worker_name}}` | 资源名称 |
| `{{worker_id}}` | 云资源 ID |
| `{{infra_id}}` | 基础设施 ID |
| `{{provider}}` | 供应商名称 |
| `{{ssh_user}}` | SSH 用户名 |
| `{{index}}` | 工作节点索引（0、1、2...） |

### 行为

- 命令在远程的 `/tmp/osm-custom/` 中运行
- 自定义命令在每个工作节点上顺序执行，跨工作节点并行执行
- 首次失败会停止该工作节点的剩余命令并跳过后置命令
- 同步下载到：`<sync-dest>/<worker_name>-<ip>/<remote_path>`

## 标志参考

| 标志 | 缩写 | 描述 |
|------|-------|-------------|
| `--flow` | `-f` | Flow 工作流名称 |
| `--module` | `-m` | Module 工作流名称 |
| `--target` | `-t` | 单个目标 |
| `--target-file` | `-T` | 包含目标的文件 |
| `--provider` | `-p` | 云供应商 |
| `--instances` | `-n` | 实例数量 |
| `--timeout` | | 扫描超时（例如 `2h`、`30m`） |
| `--auto-destroy` | | 完成后销毁基础设施 |
| `--reuse` | | 自动发现现有基础设施 |
| `--reuse-with` | | 复用特定 IP（逗号分隔） |
| `--sync-back` | | 下载工作流结果（工作流模式） |
| `--verbose-setup` | | 显示完整设置命令输出 |
| `--ansible` | | 使用 Ansible playbook 进行设置 |
| `--chunk-size` | | 每个工作节点块的目标数 |
| `--chunk-count` | | 将目标分成 N 块 |
| `--custom-cmd` | | 自定义命令（可重复） |
| `--custom-post-cmd` | | 后置命令（可重复） |
| `--sync-path` | | 要下载的远程路径（可重复） |
| `--sync-dest` | | 本地同步目录（默认：`./osm-sync-back`） |

## 成本参考

| 供应商 | 实例 | vCPU | RAM | 每小时 |
|----------|----------|------|-----|--------|
| Hetzner | cx22 | 2 | 4 GB | ~$0.007 |
| Linode | g6-standard-2 | 2 | 4 GB | $0.018 |
| DigitalOcean | s-2vcpu-4gb | 2 | 4 GB | $0.02232 |
| AWS | t3.medium | 2 | 4 GB | $0.0416 |
| GCP | n1-standard-2 | 2 | 7.5 GB | $0.095 |
| Azure | Standard_B2s | 2 | 4 GB | $0.042 |

> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# CLI 接口

> Osmedeus 的命令行接口



本文档描述了 Osmedeus 的命令行接口（CLI），它是运行工作流（Workflow）和模块（Module）以及其他实用工具的主要入口点。

它提供了每个命令的使用示例和说明。以下是所有可用命令：

```bash
j3ssie ▶ osmedeus -h

用法：
  osmedeus [flags]
  osmedeus [command]

可用命令：
  agent       以交互方式运行 ACP agent
  assets      查询并列出已发现的资产（Asset）
  client      与远程 osmedeus 服务器交互
  cloud       云基础设施管理命令
  completion  为指定 shell 生成自动补全脚本
  config      管理 osmedeus 配置（Configuration）
  db          数据库管理命令
  eval        评估脚本（'func eval' 的简写）
  function    执行和测试实用函数（Function）
  health      检查并修复环境健康状态（'osmedeus install validate' 的别名）
  help        关于任何命令的帮助
  install     安装工作流（Workflow）、基础文件夹或二进制文件
  run         执行工作流（Workflow）
  scan        执行工作流（'run' 的别名）
  serve       启动 Osmedeus Web 服务器
  snapshot    导出和导入项目空间（Workspace）快照（Snapshot）
  uninstall   移除 Osmedeus 安装（基础文件夹、项目空间和二进制文件）
  update      将 osmedeus 更新到最新版本
  version     打印版本信息
  worker      用于分布式扫描的工作节点命令
  workflow    管理工作流（Workflow）
```

## 1. Osmedeus Run (别名：scan)

对一个或多个目标执行工作流（Workflow）




  

  


```bash

▷ 示例
  # 针对单个目标运行
  osmedeus run -f recon-workflow -t example.com

  # 针对多个目标运行
  osmedeus run -m simple-module -t target1.com -t target2.com

  # 通过标准输入（stdin）输入并设置并发数
  cat list-of-urls.txt | osmedeus run -m simple-module --concurrency 10

  # 组合多种输入方式
  echo "extra.com" | osmedeus run -m simple-module -t main.com -T more-targets.txt

  # 使用自定义参数运行
  osmedeus run -m simple-module -t example.com --params 'threads=20'

  # 使用自定义基础文件夹运行
  osmedeus run --base-folder /opt/osmedeus-base -f recon-workflow -t example.com

  # 设置超时（超时则取消）
  osmedeus run -m recon -t example.com --timeout 2h

  # 每小时重复运行一次
  osmedeus run -m recon -t example.com --repeat --repeat-wait-time 1h

  # 按顺序运行多个模块
  osmedeus run -m subdomain -m portscan -m vulnscan -t example.com

  # 组合超时和重复模式
  osmedeus run -m recon -t example.com --timeout 3h --repeat --repeat-wait-time 30m

  # 空运行模式（预览而不执行）
  osmedeus run -m recon -t example.com --dry-run

  # 从标准输入运行模块（管道输入 YAML）
  cat module.yaml | osmedeus run --std-module -t example.com

  # 从 YAML/JSON 文件加载参数
  osmedeus run -m recon -t example.com --params-file params.yaml

  # 自定义项目空间路径
  osmedeus run -m recon -t example.com --workspace /custom/workspace

  # 跳过启发式检查
  osmedeus run -m recon -t example.com --heuristics-check none

  # 从文件读取目标并设置并发
  osmedeus run -m recon -T targets.txt --concurrency 5

▷ 从 URL 加载模块
  # 从 URL 运行模块
  osmedeus run --module-url https://example.com/module.yaml -t example.com

  # 从 GitHub（公开仓库）运行模块
  osmedeus run --module-url https://raw.githubusercontent.com/user/repo/main/module.yaml -t example.com

  # 从私有 GitHub 仓库运行模块（需要 GH_TOKEN 或 GITHUB_API_KEY）
  osmedeus run --module-url https://github.com/user/private-repo/blob/main/module.yaml -t example.com

▷ 分块模式（将大型目标文件拆分到多台机器）
  # 查看目标文件的分块信息
  osmedeus run -m recon -T targets.txt --chunk-size 100

  # 运行指定分块（从 0 开始索引）
  osmedeus run -m recon -T targets.txt --chunk-size 100 --chunk-part 2

  # 将目标文件分成 4 个等份，并运行第 0 块
  osmedeus run -m recon -T targets.txt --chunk-count 4 --chunk-part 0

  # 跨机器分布式处理
  osmedeus run -m recon -T targets.txt --chunk-size 250 --chunk-part 0  # 机器 1
  osmedeus run -m recon -T targets.txt --chunk-size 250 --chunk-part 1  # 机器 2

▷ 队列模式（延迟执行，稍后处理）
  # 将运行任务加入队列，稍后处理
  osmedeus run --queue -m recon -t example.com

  # 从文件读取目标并加入队列
  osmedeus run --queue -m recon -T targets.txt

  # 处理队列中的任务（'osmedeus worker queue run' 的别名）
  osmedeus run --queue-run

  # 设置并发数处理队列任务
  osmedeus run --queue-run --concurrency 3

▷ 模块排除
  # 从流程中排除特定模块（精确匹配，可重复）
  osmedeus run -f general -t example.com -x subdomain-enum

  # 排除多个模块
  osmedeus run -f general -t example.com -x subdomain-enum -x portscan

  # 模糊排除名称包含子字符串的模块
  osmedeus run -f general -t example.com -X vuln

  # 组合精确排除和模糊排除
  osmedeus run -f general -t example.com -x portscan -X brute

▷ Webhook 模式（注册触发器而非立即运行）
  # 为模块注册 webhook 触发器
  osmedeus run --as-webhook -m recon -t example.com

  # 使用身份验证密钥注册
  osmedeus run --as-webhook -m recon -t example.com --webhook-auth-key my-secret-key

  # 注册流程 webhook
  osmedeus run --as-webhook -f general -t example.com --webhook-auth-key my-secret-key

▷ Cron 调度模式（创建周期性调度而非立即运行）
  # 创建 cron 调度（每天凌晨 2 点）
  osmedeus run --as-cron '0 2 * * *' -m recon -t example.com

  # 调度流程每 6 小时运行一次
  osmedeus run --as-cron '0 */6 * * *' -f general -t example.com

  # 为多个目标创建调度（每周一）
  osmedeus run --as-cron '0 0 * * 1' -m recon -T targets.txt
```

## 2. Osmedeus Function Eval

执行和测试工作流（Workflow）中可用的实用函数（Function）





```bash
▶ 子命令
  • list       - 列出所有可用函数
  • eval (e)   - 评估脚本并支持模板渲染

▷ 示例
  # 列出所有可用函数
  osmedeus func list

  # 评估一个简单函数
  osmedeus func eval 'trim("  hello  ")'

  # eval 的短别名
  osmedeus func e 'log_info("Hello World")'

  # 使用目标变量
  osmedeus func e 'fileExists("{{target}}")' -t /tmp/test.txt

  # 打印带语法高亮的 Markdown 文件
  osmedeus func e 'print_markdown_from_file("README.md")'

  # 多行脚本与变量
  osmedeus func e 'var x = trim("  test  "); log_info(x); x'

  # 发起 HTTP 请求
  osmedeus func e 'httpRequest("https://api.example.com", "GET", {}, "")'

  # 使用自定义参数
  osmedeus func e 'log_info("{{host}}:{{port}}")' --params 'host=localhost' --params 'port=8080'

  # 使用 -f 标志为文件参数启用 shell 路径自动补全
  osmedeus func e -f trim "  hello world  "
  osmedeus func e -f fileExists /tmp
  osmedeus func e -t example.com -f log_info "Processing {{target}}"

  # 使用 SQL 查询数据库
  osmedeus func e 'db_select("SELECT severity, COUNT(*) FROM vulnerabilities GROUP BY severity", "markdown")'

  # 从数据库查询过滤后的资产
  osmedeus func e 'db_select_assets_filtered("example.com", 200, "subdomain", "jsonl")'

  # 从标准输入读取脚本
  echo 'log_info("hello")' | osmedeus func e --stdin

  # 另一种标准输入语法
  echo 'trim("  test  ")' | osmedeus func e -

▷ 批量处理
  # 从文件处理多个目标
  osmedeus func e 'log_info("Processing: " + target)' -T targets.txt

  # 从文件加载函数并处理目标
  osmedeus func e --function-file check.js -T targets.txt

  # 设置并发数
  osmedeus func e 'httpGet("https://" + target)' -T targets.txt -c 10

  # 结合参数
  osmedeus func e 'log_info(prefix + target)' -T targets.txt --params 'prefix=test_' -c 5
```

## 3. 安装命令

从各种来源安装工作流（Workflow）、基础文件夹或二进制文件的一行命令。

```bash
◆ 描述
  从各种来源安装工作流、基础文件夹或二进制文件。

▶ 子命令
  • workflow  - 从 Git URL、ZIP URL 或本地 ZIP 文件安装工作流
  • base      - 安装基础文件夹（备份并恢复数据库）
  • binary    - 从注册表安装二进制文件
  • env       - 将二进制文件路径添加到 shell 配置
  • validate  - 检查并修复环境健康状态

▷ 示例
  # 列出可用二进制文件（直接获取模式）
  osmedeus install binary --list-registry-direct-fetch

  # 列出可用二进制文件（Nix 构建模式）
  osmedeus install binary --list-registry-nix-build

  # 安装指定二进制文件
  osmedeus install binary --name nuclei --name httpx

  # 安装所有必需的二进制文件
  osmedeus install binary --all

  # 安装所有二进制文件（包括可选的）
  osmedeus install binary --all --install-optional

  # 检查二进制文件是否已安装
  osmedeus install binary --all --check

  # 安装 Nix 包管理器
  osmedeus install binary --nix-installation

  # 通过 Nix 安装二进制文件
  osmedeus install binary --name nuclei --nix-build-install

  # 通过 Nix 安装所有二进制文件
  osmedeus install binary --all --nix-build-install

  # 从 Git、ZIP URL 或本地 ZIP 文件安装工作流
  osmedeus install workflow https://github.com/user/osmedeus-workflows.git
  osmedeus install workflow http://<custom-host>/workflow-osmedeus.zip
  osmedeus install workflow local-file-workflow-osmedeus.zip

  # 从 Git、ZIP URL 或本地 ZIP 文件安装基础文件夹
  osmedeus install base https://github.com/user/osmedeus-base.git
  osmedeus install base http://<custom-host>/osmedeus-base.zip
  osmedeus install base local-file-osmedeus-base.zip
```

## 4. 数据库管理

查看和管理数据库，包括清理和模式操作。





```bash
◆ 描述
  列出所有数据库表及其行数，或从指定表中列出记录（支持分页）。

▶ 子命令
  • list (ls)  - 列出数据库表和行数（默认）
  • seed       - 用示例数据填充数据库
  • clean      - 从数据库中删除所有数据
  • migrate    - 运行数据库迁移
  • index      - 将文件系统中的资源索引到数据库

▶ 选项（list）
  -t, --table         要列出记录的表名
  --offset            要跳过的记录数（默认：0）
  --limit             返回的最大记录数（默认：50）
  --list-columns      列出指定表的所有可用列
  --exclude-columns   从输出中排除的列名（逗号分隔）
  --columns           要显示的列（逗号分隔）
  --all               显示所有列，包括隐藏列
  --search            在所有列中搜索子字符串
  --where             过滤记录（key=value 格式，可重复）
  --refresh           自动刷新间隔（例如 5s, 1m）
  --clear             清空指定表的所有记录（需要 --force）

▶ 有效表名
  runs, step_results, artifacts, assets, event_logs, schedules, workspaces, vulnerabilities

▷ 列表示例
  # 列出所有表及其行数
  osmedeus db list

  # 列出 runs 表中的记录
  osmedeus db list -t runs

  # 列出 assets 表的可用列
  osmedeus db list -t assets --list-columns

  # 列出资产，排除指定列
  osmedeus db list -t assets --exclude-columns id,created_at,updated_at

  # 分页列出资产
  osmedeus db list -t assets --offset 0 --limit 10

  # 获取下一页结果
  osmedeus db list -t assets --offset 10 --limit 10

  # 每 5 秒自动刷新
  osmedeus db list -t runs --refresh 5s

  # JSON 输出
  osmedeus db list -t runs --json

▷ 填充示例
  # 用示例数据填充数据库
  osmedeus db seed

▷ 清理示例
  # 从数据库中删除所有数据（需要 --force）
  osmedeus db clean --force

  # 删除所有数据，包括项目空间文件
  osmedeus db clean --force --clean-ws

▷ 迁移示例
  # 运行数据库迁移
  osmedeus db migrate

▷ 索引示例
  # 将文件系统中的工作流索引到数据库
  osmedeus db index workflow

  # 强制重新索引所有工作流
  osmedeus db index workflow --force
```

## 5. 队列管理

将任务加入队列以便稍后处理，并以可控的并发数执行。任务可以通过 `osmedeus run` 的 `--queue` 标志加入队列，或直接通过 `osmedeus worker queue new` 加入。

```bash
▶ 完整工作流
  # 步骤 1：将任务加入队列
  osmedeus run --queue -m recon -t example.com
  osmedeus run --queue -m recon -T targets.txt

  # 步骤 2：列出队列中的任务
  osmedeus worker queue list

  # 步骤 2b：以 JSON 格式列出（用于脚本）
  osmedeus worker queue list --json

  # 步骤 3：处理所有队列中的任务
  osmedeus worker queue run

  # 步骤 3b：以更高的并发数处理
  osmedeus worker queue run --concurrency 3

  # 步骤 3c：使用自定义 Redis URL 处理
  osmedeus worker queue run --redis-url redis://localhost:6379

▶ 快捷方式
  # 一步完成队列和处理（'worker queue run' 的别名）
  osmedeus run --queue-run
  osmedeus run --queue-run --concurrency 3

▶ 直接创建队列
  # 将模块运行加入队列
  osmedeus worker queue new -m recon -t example.com

  # 将流程运行（从文件读取目标）加入队列
  osmedeus worker queue new -f general -T targets.txt

  # 带参数加入队列
  osmedeus worker queue new -m recon -t example.com -p 'threads=20'
```
## 6. Server

启动 Osmedeus Web 服务器，提供用于管理运行、工作流和设置的 REST API 端点。

```bash
▷ 示例
  # 使用默认设置启动服务器
  osmedeus serve

  # 在自定义端口上启动服务器
  osmedeus serve --port 8080

  # 启动服务器时不进行身份验证（仅限开发环境）
  osmedeus serve -A

  # 在特定主机上启动服务器，不进行身份验证
  osmedeus serve --host 127.0.0.1 --port 8811 -A

  # 作为分布式主节点启动
  osmedeus serve --master

  # 启动服务器时不进行队列轮询
  osmedeus serve --no-queue-polling
```

## 7. Worker（分布式模式）

用于在分布式扫描模式下管理工作节点的命令。Worker 连接到 Redis 并从主节点处理任务。

```bash
▶ 子命令
  • join    - 加入分布式 Worker 池
  • status  - 显示 Worker 池状态（别名：ls）
  • set     - 更新 Worker 字段（别名、公网 IP、SSH 启用、SSH 密钥路径）
  • eval    - 使用分布式钩子评估函数表达式
  • queue   - 管理和处理队列中的任务（list、new、run）

▷ Join 示例
  # 使用 osm-settings.yaml 中的设置加入
  osmedeus worker join

  # 使用特定的 Redis URL 加入
  osmedeus worker join --redis-url redis://user:pass@localhost:6379/0

  # 加入并自动检测公网 IP
  osmedeus worker join --get-public-ip

▷ Status
  # 以表格形式显示 Worker 状态
  osmedeus worker status

  # 以 JSON 格式输出 Worker 信息
  osmedeus worker status --json

▷ 设置 Worker 字段
  # 为 Worker 设置别名
  osmedeus worker set <worker-id> alias scanner-1

  # 设置公网 IP
  osmedeus worker set scanner-1 public-ip 203.0.113.10

  # 启用 SSH
  osmedeus worker set scanner-1 ssh-enabled true

  # 使用自定义 Redis URL
  osmedeus worker set <worker-id> alias prod-1 --redis-url redis://localhost:6379

▷ Eval（一次性执行，带分布式钩子）
  # 使用分布式钩子执行简单函数
  osmedeus worker eval 'log_info("hello from worker eval")' --redis-url redis://localhost:6379

  # 将调用路由到主节点
  osmedeus worker eval 'run_on_master("func", "log_info(\"routed via redis\")")' --redis-url redis://localhost:6379

  # 带目标变量
  osmedeus worker eval 'log_info("hello")' -t example.com --redis-url redis://localhost:6379

  # 从标准输入读取脚本
  echo 'run_on_master("func", "db_import_sarif(\"ws\", \"/path/f.sarif\")")' | osmedeus worker eval --stdin --redis-url redis://localhost:6379
```

## 8. Webhook 触发器

将运行注册为可通过 HTTP 请求调用的 Webhook 触发器。这允许外部系统（CI/CD、监控工具等）按需触发扫描。

Webhook 触发器要求服务器在 `osm-settings.yaml` 中设置 `enable_trigger_via_webhook: true`。

```bash
▶ 设置
  # 步骤 1：注册一个 Webhook 触发器
  osmedeus run --as-webhook -m recon -t example.com

  # 使用身份验证密钥注册以增强安全性
  osmedeus run --as-webhook -m recon -t example.com --webhook-auth-key my-secret-key

▶ 管理
  # 列出所有已注册的 Webhook 触发器
  osmedeus worker webhooks

  # 以 JSON 格式列出（适用于脚本）
  osmedeus --json worker webhooks

▶ 通过 HTTP 触发
  # 触发 Webhook（GET - 无覆盖）
  curl https://your-server/osm/api/webhook-runs/<webhook-uuid>/trigger

  # 使用身份验证密钥触发
  curl https://your-server/osm/api/webhook-runs/<webhook-uuid>/trigger?key=my-secret-key

  # 通过 POST 触发，覆盖目标
  curl -X POST https://your-server/osm/api/webhook-runs/<webhook-uuid>/trigger \
    -H 'Content-Type: application/json' \
    -d '{"target": "new-target.com"}'

  # 通过 POST 触发，覆盖工作流
  curl -X POST https://your-server/osm/api/webhook-runs/<webhook-uuid>/trigger \
    -H 'Content-Type: application/json' \
    -d '{"target": "example.com", "module": "subdomain"}'

▶ 配置
  # 在服务器上启用 Webhook 触发
  osmedeus config set server.enable_trigger_via_webhook true
```

## 9. Assets



从数据库中查询和列出已发现的资产。这是 `osmedeus db ls -t assets` 的快捷方式，支持模糊搜索、来源/类型过滤和资产统计。

```bash
▷ 示例
  # 列出所有资产（默认列）
  osmedeus assets

  # 跨资产字段进行模糊搜索
  osmedeus assets example.com

  # 按项目空间过滤
  osmedeus assets -w myworkspace

  # 按来源过滤
  osmedeus assets --source httpx

  # 按资产类型过滤
  osmedeus assets --type web

  # 组合过滤
  osmedeus assets --source httpx --type web

  # 显示资产统计信息
  osmedeus assets --stats

  # 按项目空间过滤统计信息
  osmedeus assets --stats -w myworkspace

  # 带分页
  osmedeus assets example.com --limit 100

  # JSON 输出
  osmedeus assets example.com --json

  # 自定义列
  osmedeus assets --columns "asset_value,url,status_code"
```

| 标志                | 简写 | 默认值 | 描述                                  |
| ------------------- | ---- | ------ | ------------------------------------- |
| `--workspace`       | `-w` | `""`   | 按项目空间名称过滤                    |
| `--source`          | —    | `""`   | 按来源过滤（例如 httpx、subfinder）   |
| `--type`            | —    | `""`   | 按资产类型过滤（例如 web、subdomain） |
| `--stats`           | —    | `false`| 显示资产统计信息                      |
| `--limit`           | —    | `50`   | 返回的最大记录数                      |
| `--offset`          | —    | `0`    | 跳过的记录数（用于分页）              |
| `--columns`         | —    | `""`   | 要显示的列（逗号分隔）                |
| `--exclude-columns` | —    | `""`   | 要从输出中排除的列                    |
| `--all`             | —    | `false`| 显示所有列，包括隐藏列                |

## 10. Snapshot

将项目空间快照导出和导入为压缩的 ZIP 存档。

```bash
▷ 导出
  # 将项目空间导出为快照
  osmedeus snapshot export <workspace-name>

  # 使用自定义输出路径导出
  osmedeus snapshot export <workspace-name> -o /path/to/output.zip

▷ 导入
  # 从本地文件导入
  osmedeus snapshot import /path/to/snapshot.zip

  # 从 URL 导入
  osmedeus snapshot import https://example.com/snapshot.zip

  # 仅导入文件（跳过数据库导入）
  osmedeus snapshot import /path/to/snapshot.zip --skip-db

  # 导入时不显示确认提示
  osmedeus snapshot import /path/to/snapshot.zip --force

▷ 列出
  # 列出可用的快照
  osmedeus snapshot list
```

## 11. Client（远程服务器）

通过 REST API 与远程 osmedeus 服务器交互。需要设置环境变量以进行连接。

```bash
▶ 环境设置
  export OSM_REMOTE_URL="http://localhost:8002"
  export OSM_REMOTE_AUTH_KEY="your-api-key"

▶ 子命令
  • fetch  - 从服务器获取数据（运行、资产、漏洞等）
  • run    - 创建或取消运行
  • exec   - 远程执行函数

▷ Fetch 示例
  # 获取资产（默认表格）
  osmedeus client fetch
  osmedeus client fetch -t assets -w example.com

  # 获取运行
  osmedeus client fetch --table runs
  osmedeus client fetch -t runs --status running

  # 按严重性过滤获取漏洞
  osmedeus client fetch -t vulnerabilities --severity critical

  # 获取步骤结果
  osmedeus client fetch -t step_results

  # 分页
  osmedeus client fetch -t assets --limit 50 --offset 100

  # JSON 输出
  osmedeus client --json fetch -t runs

▷ Run 示例
  # 创建 Flow 运行
  osmedeus client run -f basic-recon -T example.com

  # 创建模块运行
  osmedeus client run -m subdomain -T example.com

  # 按 ID 取消运行
  osmedeus client run --cancel abc123-run-uuid

  # JSON 输出
  osmedeus client --json run -f recon -T example.com

▷ Exec 示例
  # 执行简单函数
  osmedeus client exec 'log_info("Hello from remote")'

  # 带目标变量
  osmedeus client exec -t example.com 'fileExists("{{target}}/output.txt")'

  # 使用 --script 标志
  osmedeus client exec -s 'trim("  hello  ")'

  # JSON 输出
  osmedeus client --json exec 'trim("  test  ")'
```

## 12. 卸载

移除 Osmedeus 安装，包括基础文件夹、配置，并可选择移除项目空间数据。

```bash
▶ 将被移除的内容
  • ~/osmedeus-base         - 设置、工作流、二进制文件、数据
  • ~/.osmedeus             - 初始化标记
  • osmedeus 二进制文件     - 从 PATH 中移除（最多 3 个位置）

  使用 --clean 时：
  • ~/workspaces-osmedeus   - 所有扫描结果和项目空间数据

▷ 示例
  # 预览将被移除的内容（显示确认提示）
  osmedeus uninstall

  # 卸载但不删除项目空间（保留扫描结果）
  osmedeus uninstall --force

  # 完全卸载，包括所有扫描数据
  osmedeus uninstall --force --clean
```

## 13. 配置管理

使用点符号管理 osmedeus 配置设置。

```bash
▶ 子命令
  • clean  - 将配置重置为默认值
  • set    - 设置配置值
  • view   - 查看配置值
  • list   - 列出配置值

▷ Set 示例
  osmedeus config set server.port 9000
  osmedeus config set server.username admin
  osmedeus config set server.password "d8506b99a052e797f73d1dab"
  osmedeus config set server.jwt.secret_signing_key "d8506b99a052e797f73d1dab"
  osmedeus config set scan_tactic.default 20
  osmedeus config set global_vars.github_token ghp_xxx
  osmedeus config set notification.enabled true

▷ View 示例
  # 精确键查找
  osmedeus config view server.port
  osmedeus config view server.username
  osmedeus config view server.jwt.secret_signing_key --redact

  # 通配符模式搜索（需要 --force）
  osmedeus config view 'server.*' --force
  osmedeus config view 'database.*' --force
  osmedeus config view '*password*' --force
  osmedeus config view 'server.*' --force --redact

▷ List 和 Clean
  # 列出所有配置值
  osmedeus config list

  # 列出包括密钥
  osmedeus config list --show-secrets

  # 将配置重置为默认值（先备份现有配置）
  osmedeus config clean
```

## 14. 云基础设施

为分布式扫描配置和管理云基础设施。支持多个供应商（DigitalOcean、AWS、GCP、Linode、Azure）。

```bash
▶ 子命令
  • config  - 管理云配置（set、list）
  • create  - 创建云基础设施
  • list    - 列出活跃的云基础设施
  • destroy - 销毁云基础设施
  • run     - 在云基础设施上运行工作流

▷ 配置
  # 设置云供应商
  osmedeus cloud config set defaults.provider digitalocean

  # 设置供应商凭据
  osmedeus cloud config set providers.digitalocean.token dop_v1_xxxx
  osmedeus cloud config set providers.digitalocean.region nyc1
  osmedeus cloud config set providers.digitalocean.size s-2vcpu-4gb

  # AWS 配置
  osmedeus cloud config set providers.aws.access_key_id AKIAXXXX
  osmedeus cloud config set providers.aws.secret_access_key xxxx
  osmedeus cloud config set providers.aws.region us-east-1

  # 设置限制
  osmedeus cloud config set limits.max_instances 10
  osmedeus cloud config set limits.max_hourly_spend 5.00

  # 列出所有云配置值
  osmedeus cloud config list

  # 列出包括密钥
  osmedeus cloud config list --show-secrets

▷ 基础设施管理
  # 创建云实例
  osmedeus cloud create --instances 3

  # 使用特定供应商和模式创建
  osmedeus cloud create --provider digitalocean --mode vm --instances 5

  # 列出活跃的基础设施
  osmedeus cloud list

  # 按 ID 销毁基础设施
  osmedeus cloud destroy <infrastructure-id>

▷ 云运行
  # 在云基础设施上运行工作流
  osmedeus cloud run -f general -t example.com --instances 3
```

## 15. 工作流管理

列出、查看、搜索和验证可用的工作流。

```bash
▶ 子命令
  • list (ls)   - 列出可用的工作流（默认）
  • show (view) - 显示工作流详情
  • validate    - 验证和检查工作流
  • install     - 从源安装工作流

▷ 列出工作流
  # 列出所有可用的工作流
  osmedeus workflow list

  # 按名称、描述或标签搜索工作流
  osmedeus workflow ls recon
  osmedeus workflow ls --search subdomain

  # 按标签过滤
  osmedeus workflow ls --tags recon,fast

  # 显示标签列
  osmedeus workflow ls --show-tags

  # 显示使用信息
  osmedeus workflow ls --usage

  # 显示有解析错误的工作流（详细模式）
  osmedeus workflow ls -v

▷ 显示工作流详情
  # 以表格格式显示工作流详情
  osmedeus workflow show general

  # 显示详细的变量描述
  osmedeus workflow show general -v

  # 显示原始 YAML 并带有语法高亮
  osmedeus workflow show general --yaml

▷ 验证 / 检查工作流
  # 按名称验证工作流
  osmedeus workflow validate my-module

  # 验证 YAML 文件
  osmedeus workflow lint ./my-workflow.yaml

  # 验证文件夹中的所有工作流
  osmedeus workflow validate /path/to/workflows/

  # 在第一个失败时停止
  osmedeus workflow validate . --fail-fast

  # CI 模式（如果发现问题则退出并返回错误码）
  osmedeus workflow lint my-workflow.yaml --check

  # JSON 输出格式
  osmedeus workflow lint my-workflow.yaml --format json

  # GitHub Actions 注释格式
  osmedeus workflow lint my-workflow.yaml --format github

  # 禁用特定的 lint 规则
  osmedeus workflow validate . --disable unused-variable

  # 按最低严重性过滤
  osmedeus workflow lint my-workflow.yaml --severity error

▷ 安装工作流
  # 从 Git URL 安装
  osmedeus workflow install https://github.com/user/osmedeus-workflows.git

  # 从预设安装（使用 OSM_WORKFLOW_URL 或默认值）
  osmedeus workflow install --preset
```

## 16. 更新

将 osmedeus 更新到 GitHub 发布的最新版本。

```bash
▷ 示例
  # 检查更新但不安装
  osmedeus update --check

  # 更新到最新版本
  osmedeus update

  # 跳过确认提示
  osmedeus update --yes

  # 强制重新安装当前版本
  osmedeus update --force

  # 更新到特定版本
  osmedeus update --version v5.1.0
```

## 17. 版本

打印版本和构建信息。

```bash
▷ 示例
  # 显示版本信息
  osmedeus version

  # JSON 输出
  osmedeus --json version
```

## 18. Agent（ACP）

从终端交互式运行 ACP（Agent Communication Protocol）代理。agent 命令会生成一个外部 AI 编码代理作为子进程，并通过 ACP 进行通信。

```bash
◆ 描述
  交互式运行 ACP 代理。支持多个代理后端。

▶ 可用代理
  • claude-code  - Claude Code（默认）— npx @zed-industries/claude-code-acp@latest
  • codex        - OpenAI Codex — npx @zed-industries/codex-acp
  • opencode     - OpenCode — opencode acp
  • gemini       - Google Gemini — gemini --experimental-acp

▷ 示例
  # 使用默认代理运行（claude-code）
  osmedeus agent "分析 /tmp/output 中的扫描结果"

  # 使用特定代理
  osmedeus agent --agent codex "审查此代码是否存在漏洞"

  # 列出可用代理
  osmedeus agent --list

  # 设置工作目录
  osmedeus agent --cwd /path/to/project "总结发现"

  # 自定义超时（默认：30m）
  osmedeus agent --timeout 1h "进行彻底分析"

  # 从标准输入读取消息
  echo "分析此输出" | osmedeus agent --stdin

  # 使用短横线简写进行管道传输
  cat prompt.txt | osmedeus agent -
```

| 标志        | 默认值       | 描述                                      |
| ----------- | ------------ | ----------------------------------------- |
| `--agent`   | `claude-code`| 要使用的代理（参见 `--list` 获取可用代理）|
| `--cwd`     | 当前目录     | 代理的工作目录                            |
| `--stdin`   | `false`      | 从标准输入读取消息                        |
| `--timeout` | `30m`        | 超时时间（例如 30m、1h）                  |
| `--list`    | `false`      | 列出可用代理                              |

## 完整使用示例

请参阅 [完整使用示例](/reference/cli-references) 获取完整的 CLI 使用示例。

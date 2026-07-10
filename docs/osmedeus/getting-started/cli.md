> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# CLI 接口

> Osmedeus 的命令行接口

<Frame caption="CLI 运行进度">
  <img src="https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/cli/cli-run-progress.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=a0c448d97d10f955df1589deb2251cc6" alt="描述性替代文本" width="2704" height="1438" data-path="images/cli/cli-run-progress.png" />
</Frame>

本文档描述了 Osmedeus 的命令行接口（CLI），它是运行工作流、模块以及其他实用工具的主要入口点。

本文档提供了每个命令的使用示例和说明。以下是所有可用命令：

```bash theme={null}
j3ssie ▶ osmedeus -h

用法：
  osmedeus [flags]
  osmedeus [command]

可用命令：
  agent       以交互方式运行 ACP agent
  assets      查询并列出已发现的资产
  client      与远程 osmedeus 服务器交互
  cloud       云基础设施管理命令
  completion  为指定 shell 生成自动补全脚本
  config      管理 osmedeus 配置
  db          数据库管理命令
  eval        评估脚本（'func eval' 的简写）
  function    执行和测试实用函数
  health      检查并修复环境健康状态（'osmedeus install validate' 的别名）
  help        关于任何命令的帮助
  install     安装工作流、基础文件夹或二进制文件
  run         执行工作流
  scan        执行工作流（'run' 的别名）
  serve       启动 Osmedeus Web 服务器
  snapshot    导出和导入项目空间快照
  uninstall   移除 Osmedeus 安装（基础文件夹、项目空间和二进制文件）
  update      将 osmedeus 更新到最新版本
  version     打印版本信息
  worker      用于分布式扫描的工作节点命令
  workflow    管理工作流
```

## 1. Osmedeus Run（别名：scan）

对一个或多个目标执行工作流

<Frame caption="带有 LLM 步骤的 CLI 运行">
  <img src="https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/cli/cli-run-with-llm-step.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=966352fbdd154b60dcab8c203f54e9b7" alt="描述性替代文本" width="4336" height="2650" data-path="images/cli/cli-run-with-llm-step.png" />
</Frame>

<Columns cols={2}>
  <Frame caption="CLI 运行进度">
    <img src="https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/cli/cli-run-with-verbose-output.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=a254d87b37a7ec88651d9bbf8436be22" alt="描述性替代文本" width="3824" height="2366" data-path="images/cli/cli-run-with-verbose-output.png" />
  </Frame>

  <Frame caption="CLI 运行进度">
    <img src="https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/cli/cli-run-with-ci-friendly-output.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=c2d2b2023238bc659a30067eeffadfd0" alt="描述性替代文本" width="3824" height="2366" data-path="images/cli/cli-run-with-ci-friendly-output.png" />
  </Frame>
</Columns>

```bash theme={null}

▷ 示例
  # 针对单个目标运行
  osmedeus run -f recon-workflow -t example.com

  # 针对多个目标运行
  osmedeus run -m simple-module -t target1.com -t target2.com

  # 通过 stdin 输入并设置并发数
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

  # 组合超时与重复模式
  osmedeus run -m recon -t example.com --timeout 3h --repeat --repeat-wait-time 30m

  # 试运行模式（预览而不执行）
  osmedeus run -m recon -t example.com --dry-run

  # 从 stdin 运行模块（管道传入 YAML）
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

  # 运行指定分块（从 0 开始）
  osmedeus run -m recon -T targets.txt --chunk-size 100 --chunk-part 2

  # 将目标文件等分为 4 块，运行第 0 块
  osmedeus run -m recon -T targets.txt --chunk-count 4 --chunk-part 0

  # 跨机器分布式处理
  osmedeus run -m recon -T targets.txt --chunk-size 250 --chunk-part 0  # 机器 1
  osmedeus run -m recon -T targets.txt --chunk-size 250 --chunk-part 1  # 机器 2

▷ 队列模式（延迟执行，稍后处理）
  # 将运行任务加入队列，稍后处理
  osmedeus run --queue -m recon -t example.com

  # 将文件中的目标加入队列
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

执行和测试工作流中可用的实用函数

<Frame caption="CLI Function Eval 查询数据库">
  <img src="https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/cli/cli-func-eval-1.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=91970d6b7b8e529b79f9cc97727ef45a" alt="描述性替代文本" width="3238" height="1370" data-path="images/cli/cli-func-eval-1.png" />
</Frame>

<Frame caption="CLI Function Eval 渲染 Markdown">
  <img src="https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/cli/cli-func-eval-2.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=c94d0a37bffd99c3ee5cfbfadfd4e4b0" alt="描述性替代文本" width="4336" height="1656" data-path="images/cli/cli-func-eval-2.png" />
</Frame>

```bash theme={null}
▶ 子命令
  • list       - 列出所有可用函数
  • eval (e)   - 评估脚本并渲染模板

▷ 示例
  # 列出所有可用函数
  osmedeus func list

  # 评估一个简单函数
  osmedeus func eval 'trim("  hello  ")'

  # eval 的短别名
  osmedeus func e 'log_info("Hello World")'

  # 使用目标变量
  osmedeus func e 'fileExists("{{target}}")' -t /tmp/test.txt

  # 打印带有语法高亮的 Markdown 文件
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

  # 从 stdin 读取脚本
  echo 'log_info("hello")' | osmedeus func e --stdin

  # 另一种 stdin 语法
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

从多种来源安装工作流、基础文件夹或二进制文件的一行命令。

```bash theme={null}
◆ 描述
  从多种来源安装工作流、基础文件夹或二进制文件。

▶ 子命令
  • workflow  - 从 Git URL、zip URL 或本地 zip 文件安装工作流
  • base      - 安装基础文件夹（备份并恢复数据库）
  • binary    - 从注册表安装二进制文件
  • env       - 将二进制文件路径添加到 shell 配置
  • validate  - 检查并修复环境健康状态

▷ 示例
  # 列出可用二进制文件（直接获取模式）
  osmedeus install binary --list-registry-direct-fetch

  # 列出可用二进制文件（Nix 构建模式）
  osmedeus install binary --list-registry-nix-build

  # 安装特定二进制文件
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

  # 从 git、zip URL 或本地 zip 文件安装工作流
  osmedeus install workflow https://github.com/user/osmedeus-workflows.git
  osmedeus install workflow http://<custom-host>/workflow-osmedeus.zip
  osmedeus install workflow local-file-workflow-osmedeus.zip

  # 从 git、zip URL 或本地 zip 文件安装基础文件夹
  osmedeus install base https://github.com/user/osmedeus-base.git
  osmedeus install base http://<custom-host>/osmedeus-base.zip
  osmedeus install base local-file-osmedeus-base.zip
```

## 4. 数据库管理

查看和管理数据库，包括清理和模式操作。

<Frame caption="CLI 数据库列表">
  <img src="https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/cli/cli-db-list.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=7ca98e6543e673732ee4d330014f6644" alt="cli-db-list" width="4336" height="1824" data-path="images/cli/cli-db-list.png" />
</Frame>

<Frame caption="CLI 数据库表视图">
  <img src="https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/cli/cli-db-table-view.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=db001562f81a2723ed7a72a4950c0051" alt="cli-db-view" width="4336" height="2674" data-path="images/cli/cli-db-table-view.png" />
</Frame>

```bash theme={null}
◆ 描述
  列出所有数据库表及其行数，或列出特定表中的记录（支持分页）。

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
  --refresh           自动刷新间隔（例如 5s、1m）
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

  # 列出资产，排除特定列
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

```bash theme={null}
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
  # 一步完成入队和处理（'worker queue run' 的别名）
  osmedeus run --queue-run
  osmedeus run --queue-run --concurrency 3

▶ 直接创建队列
  # 将模块运行加入队列
  osmedeus worker queue new -m recon -t example.com

  # 将流程运行加入队列，目标来自文件
  osmedeus worker queue new -f general -T targets.txt

  # 带参数加入队列
  osmedeus worker queue new -m recon -t example.com -p 'threads=20'
```

## 6. 服务器

启动 Osmedeus Web 服务器，提供用于管理运行、工作流和设置的 REST API 端点。

```bash theme={null}
▷ 示例
  # 使用默认设置启动服务器
  osmedeus serve

  # 在自定义端口上启动服务器
  osmedeus serve --port 8080

  # 启动服务器时不启用身份验证（仅限开发环境）
  osmedeus serve -A

  # 在特定主机上启动服务器，不启用身份验证
  osmedeus serve --host 127.0.0.1 --port 8811 -A

  # 作为分布式主节点启动
  osmedeus serve --master

  # 启动服务器时不进行队列轮询
  osmedeus serve --no-queue-polling
```

## 7. Worker（分布式模式）

用于管理分布式扫描模式中工作节点的命令。工作节点连接到 Redis 并从主节点处理任务。

```bash theme={null}
▶ 子命令
  • join    - 加入分布式工作节点池
  • status  - 显示工作节点池状态（别名：ls）
  • set     - 更新工作节点字段（别名、公网 IP、SSH 启用、SSH 密钥路径）
  • eval    - 使用分布式钩子评估函数表达式
  • queue   - 管理和处理队列中的任务（list、new、run）

▷ Join 示例
  # 使用 osm-settings.yaml 中的设置加入
  osmedeus worker join

  # 使用特定的 Redis URL 加入
  osmedeus worker join --redis-url redis://user:pass@localhost:6379/0

  # 加入并自动检测公网 IP
  osmedeus worker join --get-public-ip

▷ 状态
  # 以表格形式显示工作节点状态
  osmedeus worker status

  # 以 JSON 格式输出工作节点信息
  osmedeus worker status --json

▷ 设置工作节点字段
  # 为工作节点设置别名
  osmedeus worker set <worker-id> alias scanner-1

  # 设置公网 IP
  osmedeus worker se
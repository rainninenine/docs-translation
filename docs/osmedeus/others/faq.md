> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 常见问题解答

> 常见问题与常见错误

收集了常见问题以及常见错误的解释和解决方法。

## 1. 通用问题

<ResponseField name="什么是 Osmedeus？">
  <Expandable title="答案">
    Osmedeus 是一个用于安全自动化的工作流引擎。它执行 YAML 定义的工作流，支持多种执行环境（主机、Docker、SSH）、调度和分布式扫描。
  </Expandable>
</ResponseField>

<ResponseField name="系统要求是什么？">
  <Expandable title="答案" defaultOpen="true">
    Osmedeus 核心引擎非常轻量，几乎可以在任何规格的系统上运行。<br />

    但是，如果你打算将其用于**侦察**（这是主要用例），建议使用现代 Linux、macOS 或带有 WSL 的 Windows 系统。<br />

    由于运行侦察会产生大量网络流量，还建议在云环境中运行 Osmedeus，**例如 VM、Compute Engine 或 EC2**，以获得最佳性能。
  </Expandable>
</ResponseField>

<ResponseField name="如何安装 Osmedeus？">
  <Expandable title="答案">
    ```bash theme={null}
    # 从源码构建
    make build

    # 安装到 $GOBIN
    make install

    # 安装安全工具
    osmedeus install binary --all
    ```
  </Expandable>
</ResponseField>

<ResponseField name="Osmedeus 是否支持 AI/LLM 集成？我可以使用 Claude Code 和 Codex 等工具吗？">
  <Expandable title="答案">
    当然可以。Osmedeus 内置了对 LLM 的支持，你可以在工作流中使用它来生成侦察报告、编写自定义脚本，甚至构建你自己的代理工作流。你可以查看 [LLM 工作流示例](/advanced/llm/) 了解其工作原理。

    请注意，使用 LLM 可能需要你拥有 LLM 提供商的 API 密钥，并且可能会根据你的使用情况产生额外费用。在工作流中使用 LLM 时，请始终监控你的使用情况和成本。

    由于 Osmedeus 是一个编排框架，你可以利用它来协调你自己的自定义 AI/LLM 工具，包括直接在 YAML 工作流中集成 Claude Code 或 OpenCode。例如，你可以设计一个自定义代理，作为定义管道的一部分调用多个工具，并将其无缝插入到你的工作流中。灵活性几乎是无限的。
  </Expandable>
</ResponseField>

<ResponseField name="Osmedeus 是否有我可以在代理中使用的 AI 技能？" defaultOpen="true">
  <Expandable title="答案">
    是的，我在 [github.com/osmedeus/osmedeus-skills](https://github.com/osmedeus/osmedeus-skills) 上构建了 [osmedeus-expert](https://github.com/osmedeus/osmedeus-skills) 技能，你可以在你的代理工具中使用它来编写 YAML 工作流、运行 CLI 命令和配置高级功能。
  </Expandable>
</ResponseField>

***

## 2. 二进制安装

<ResponseField name="为什么我需要安装外部二进制文件？">
  <Expandable title="答案" defaultOpen="true">
    Osmedeus 是一个独立的 Golang 二进制文件，本身可以完美运行。但是，当使用 Osmedeus 运行用于安全自动化的 YAML 工作流时，它通常需要调用外部工具，如 `httpx, nuclei, ffuf` 等。这些工具必须安装并在你的系统上可用，这些工作流才能正常运行。
  </Expandable>
</ResponseField>

<ResponseField name="我在安装二进制文件或运行 `osmedeus health` 时遇到错误。我该怎么办？">
  <Expandable title="答案" defaultOpen="true">
    并非注册表中列出的所有二进制文件都是每个工作流所必需的。即使缺少某些工具，你的扫描也可能正常运行。
  </Expandable>
</ResponseField>

<ResponseField name="我是否需要使用 `osmedeus install binary --all --install-optional` 安装注册表中的所有二进制文件？">
  <Expandable title="答案" defaultOpen="true">
    不需要。安装所有工具完全是可选的。注册表包含 YAML 工作流中常用的额外工具，但你只需要特定工作流所需的工具。运行基本工作流不需要安装所有工具。
  </Expandable>
</ResponseField>

<ResponseField name="我尝试了所有方法，但仍然无法安装所需的二进制文件。我该怎么办？">
  <Expandable title="答案" defaultOpen="true">
    就像我上面说的，并非注册表中列出的所有二进制文件都是每个工作流所必需的。即使缺少某些工具，你的扫描也可能正常运行。

    如果你想要理想的设置，那么我建议使用 Docker 来运行 Osmedeus 及其工作流。这可以确保满足所有依赖关系，并消除任何兼容性问题。有关更多详细信息，请参阅 [Docker 设置](/getting-started/docker-setup)。
  </Expandable>
</ResponseField>

***

## 3. 扫描执行与扫描结果

<ResponseField name="如何运行基本扫描？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    # 运行流程工作流
    osmedeus run -f general -t example.com

    # 运行模块工作流
    osmedeus run -m vulnerability-scan -t example.com
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何扫描多个目标？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    # 从命令行
    osmedeus run -f fast -t target1.com -t target2.com

    # 从文件
    osmedeus run -f fast -T targets.txt -c 5
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何设置超时？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    osmedeus run -m port-scan -t example.com --timeout 2h
    ```
  </Expandable>
</ResponseField>

<ResponseField name="扫描结果存储在哪里？">
  <Expandable title="答案" defaultOpen="true">
    结果存储在 `~/workspaces-osmedeus/<target>/` 的工作区中。
  </Expandable>
</ResponseField>

<ResponseField name="我应该把我的令牌（Github、Shodan 等）放在哪里？">
  <Expandable title="答案" defaultOpen="true">
    你只需要按照 [**本指南设置令牌**](/getting-started/basic-setup/) 操作即可。
  </Expandable>
</ResponseField>

<ResponseField name="如何设置发送通知？">
  <Expandable title="答案" defaultOpen="true">
    你只需要按照 [**本指南设置通知**](/advanced/notification-and-cdn/) 操作即可。
  </Expandable>
</ResponseField>

<ResponseField name="我在哪里可以获得实时支持？">
  <Expandable title="答案" defaultOpen="true">
    你可以加入 **`https://discord.gg/mtQG2FQsYA`** 看看是否有人能帮忙。我可能会时不时回答，但我不能保证回答每一个问题。
  </Expandable>
</ResponseField>

<ResponseField name="它支持代理吗？">
  <Expandable title="答案" defaultOpen="true">
    不，它本身不支持代理。但由于该工具的设计是运行其他第三方工具，而且很多工具默认不支持代理。我已经考虑过 proxychains，但它会使速度极慢并破坏很多东西。
  </Expandable>
</ResponseField>

<ResponseField name="为什么我的扫描卡在端口扫描？">
  <Expandable title="答案" defaultOpen="true">
    它会卡在那里，因为它遇到了 sudo 密码提示。某些特殊工具需要 *root* 权限才能运行，例如 **nmap**。请确保允许 **nmap** 在没有 sudo 密码提示的情况下运行。
  </Expandable>
</ResponseField>

<ResponseField name="为什么我的扫描（如漏洞扫描、端口扫描或内容发现）花了这么长时间？">
  <Expandable title="答案" defaultOpen="true">
    可能是因为你输入的内容非常大。想象一下尝试对 **2000 个不同的主机** 运行内容发现。这就是为什么需要很长时间。
  </Expandable>
</ResponseField>

<ResponseField name="为什么 Osmedeus 没有发现任何漏洞，即使我扫描的是故意存在漏洞的应用程序？">
  <Expandable title="答案" defaultOpen="true">
    再次强调，这很大程度上取决于你的目标。Osmedeus 在大范围目标上表现出色，而不是单个故意存在漏洞的 Web 应用程序。只需扫描一些随机的 VDP，你就会看到结果。
    它不会在故意存在漏洞的应用程序上发现任何漏洞的原因是 **vulnscan** 模块不支持它。但你随时可以自定义工作流来实现这一点。
  </Expandable>
</ResponseField>

<ResponseField name="我发现了一个很酷的新工具。你能把它添加到 Osmedeus 中吗？">
  <Expandable title="答案" defaultOpen="true">
    可以，只需按照 [**本指南**](/advanced/writing-your-first-workflow/) 将其添加到你的工作流中。
  </Expandable>
</ResponseField>

<ResponseField name="X 扫描是否运行工具 Y？">
  <Expandable title="答案" defaultOpen="true">
    1. 阅读流程和模块文件，确定某个步骤实际运行什么。
    2. 说真的，阅读流程和模块文件。
    3. 记住，你已经两次被警告要阅读流程和模块文件。
    4. 在工作流文件夹中搜索工具命令，以确认它是否被使用（例如：`rg -F 'nuclei' ~/osmedeus-base/workflows/`）。
  </Expandable>
</ResponseField>

<ResponseField name="在哪里可以找到 Web UI 的密码？">
  <Expandable title="答案" defaultOpen="true">
    请参考 [**此页面**](/getting-started/web-ui/) 启动 Web 服务器并获取凭据。你可能需要运行命令 `osmedeus config view server.password`。
  </Expandable>
</ResponseField>

<ResponseField name="如何让扫描或 Web UI 在后台运行？">
  <Expandable title="答案" defaultOpen="true">
    最简单的方法是在 `https://tmuxcheatsheet.com/` 下运行进程。除此之外，你可以设置一个服务来将 osmedeus Web 服务器作为后台进程运行。
  </Expandable>
</ResponseField>

<ResponseField name="如果 Osmedeus 发现了漏洞 X，我该怎么办？">
  <Expandable title="答案" defaultOpen="true">
    1. 阅读漏洞 X 的描述。
    2. 说真的，阅读漏洞 X 的描述。
    3. 记住，你已经两次被警告要阅读漏洞 X 的描述。
    4. 搜索漏洞 X 的名称。
    5. 手动验证漏洞 X。
    6. 还是没有结果？也许 `https://letmegooglethat.com/?q=what+is+a+vulnerability+X` 可以帮你。
  </Expandable>
</ResponseField>

<ResponseField name="Osmedeus 发现了一些易受攻击的子域名，但我无法访问它们？">
  <Expandable title="答案" defaultOpen="true">
    通常，扫描期间发现的子域名的可用性在你尝试手动验证时可能不同。这取决于目标，并且可能有所不同。
  </Expandable>
</ResponseField>

<ResponseField name="当我使用 `--debug` 标志运行时，我注意到某些命令返回了错误，退出状态为 128、255 或 -1。这是预期的吗？">
  <Expandable title="答案" defaultOpen="true">
    是的，某些命令表现出预期的退出状态是正常的，因为它们可能在特定条件下成功。但是，如果你确信原始 bash 命令应该成功但失败了，请尝试复制原始 bash 命令并调查它为什么遇到问题。
  </Expandable>
</ResponseField>

<ResponseField name="如何确定要为我的目标运行哪个工作流？">
  <Expandable title="答案" defaultOpen="true">
    你可以运行 `osmedeus workflow ls` 或 `osmedeus workflow show <workflow-name> --verbose` 查看描述，然后选择适合扫描的工作流。
  </Expandable>
</ResponseField>

<ResponseField name="扫描执行没有错误，但 UI 没有显示任何资产？">
  <Expandable title="答案" defaultOpen="true">
    这很可能是因为你执行的工作流没有生成任何资产。你可以通过检查位于 `~/workspaces-osmedeus/<target>/` 的工作区目录来验证是否创建了任何文件。

    也可能是因为工作流没有使用任何数据库实用程序函数将资产保存到数据库中。你可以检查工作流文件，看它是否使用了像 `db_import_asset` 这样的数据库实用程序函数。你还可以在 `osmedeus func ls db --example` 查看数据库相关函数的完整列表。
  </Expandable>
</ResponseField>

<ResponseField name="为什么即使我设置了通知，也没有看到任何通知？">
  <Expandable title="答案" defaultOpen="true">
    这很可能是因为你执行的工作流没有生成任何资产。你可以通过检查位于 `~/workspaces-osmedeus/<target>/` 的工作区目录来验证是否创建了任何文件。

    也可能是因为工作流没有使用任何通知实用程序函数将资产保存到通知中。你可以检查工作流文件，看它是否使用了像 `notify_telegram` 这样的通知实用程序函数。你还可以在 `osmedeus func ls noti --example` 查看通知相关函数的完整列表。
  </Expandable>
</ResponseField>

***

## 4. 工作流

<ResponseField name="流程和模块有什么区别？">
  <Expandable title="答案" defaultOpen="true">
    * **模块**：一个单一的工作流单元，包含按顺序执行的步骤。
    * **流程**：编排多个模块，允许并行执行和模块之间的依赖关系。
  </Expandable>
</ResponseField>

<ResponseField name="工作流存储在哪里？">
  <Expandable title="答案" defaultOpen="true">
    工作流存储在 `~/osmedeus-base/workflows/` 中：

    * `flows/` - 流程工作流
    * `modules/` - 模块工作流
  </Expandable>
</ResponseField>

<ResponseField name="如何创建自定义工作流？">
  <Expandable title="答案" defaultOpen="true">
    在工作流目录中创建一个 YAML 文件：

    ```yaml theme={null}
    name: my-workflow
    kind: module
    description: 我的自定义工作流
    params:
      - name: target
        required: true
    steps:
      - name: scan-target
        type: bash
        command: nmap {{target}}
    ```
  </Expandable>
</ResponseField>

<ResponseField name="有哪些步骤类型可用？">
  <Expandable title="答案" defaultOpen="true">
    | 类型             | 描述                             |
    | ---------------- | --------------------------------------- |
    | `bash`           | 执行 shell 命令                  |
    | `function`       | 执行 JavaScript 实用程序函数    |
    | `parallel-steps` | 并发运行步骤                  |
    | `foreach`        | 遍历项目                      |
    | `remote-bash`    | 在 Docker 中或通过 SSH 执行            |
    | `http`           | 发起 HTTP 请求                      |
    | `llm`            | 执行 LLM API 调用                   |
    | `agent`          | 带工具调用的代理 LLM 执行 |
  </Expandable>
</ResponseField>

***

## 5. API 与服务器

<ResponseField name="如何启动 API 服务器？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    osmedeus server
    ```

    服务器默认在端口 8002 上启动。
  </Expandable>
</ResponseField>

<ResponseField name="我遇到了 `failed to run database migrations` 错误。我该怎么办？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    $ osmedeus server
    2026-02-16T22:59:11+07:00 ERROR Failed to create server {"error": "failed to run database migrations: failed to create index: SQL logic error: no such column: webhook_uuid (1)"}
    Error: failed to run database migrations: failed to create index: SQL logic error: no such column: ... (1)
    ```

    像这样的错误意味着数据库模式已过时，服务器无法启动。要修复此问题，你可以运行 `osmedeus db clean --force` 来清理数据库，然后重新启动服务器。这将重置你的数据库，因此在运行命令之前请确保备份任何重要数据。
  </Expandable>
</ResponseField>

<ResponseField name="如何通过 API 进行身份验证？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    # 获取 JWT 令牌
    curl -X POST http://localhost:8002/osm/api/login       -H "Content-Type: application/json"       -d '{"username": "osmedeus", "password": "admin"}'

    # 使用令牌
    curl http://localhost:8002/osm/api/workflows       -H "Authorization: Bearer <token>"
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何禁用身份验证？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    osmedeus server --no-auth
    ```
  </Expandable>
</ResponseField>

<ResponseField name="我可以使用 API 密钥代替 JWT 吗？">
  <Expandable title="答案" defaultOpen="true">
    可以，在服务器配置中启用 API 密钥身份验证。然后使用 `X-API-Key` 标头代替 `Authorization: Bearer`。
  </Expandable>
</ResponseField>

***

## 6. 调度

<ResponseField name="如何安排定期扫描？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    # 通过 CLI（创建 cron 计划）
    osmedeus run -f subdomain-enum -t example.com --schedule "0 2 * * *"

    # 通过 API
    curl -X POST http://localhost:8002/osm/api/schedules       -H "Authorization: Bearer $TOKEN"       -H "Content-Type: application/json"       -d '{
        "name": "daily-scan",
        "workflow_name": "subdomain-enum",
        "target": "example.com",
        "schedule": "0 2 * * *"
      }'
    ```
  </Expandable>
</ResponseField>

<ResponseField name="使用什么 cron 格式？">
  <Expandable title="答案" defaultOpen="true">
    标准 5 字段 cron：`分钟 小时 日 月 星期`

    示例：

    * `0 2 * * *` - 每天凌晨 2 点
    * `0 0 * * 0` - 每周日
    * `*/30 * * * *` - 每 30 分钟
  </Expandable>
</ResponseField>

***

## 7. 运行器

<ResponseField name="有哪些运行器可用？">
  <Expandable title="答案" defaultOpen="true">
    | 运行器   | 描述                        |
    | -------- | ---------------------------------- |
    | `host`   | 在本地机器上执行（默认） |
    | `docker` | 在 Docker 容器中执行       |
    | `ssh`    | 通过 SSH 在远程机器上执行 |
  </Expandable>
</ResponseField>

<ResponseField name="如何在 Docker 中运行扫描？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    osmedeus run -m port-scan -t example.com --runner docker --docker-image osmedeus/osmedeus:latest
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何在远程主机上运行扫描？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    osmedeus run -m port-scan -t example.com --runner ssh --ssh-host worker.example.com
    ```
  </Expandable>
</ResponseField>

***

## 8. 分布式模式

<ResponseField name="如何设置分布式扫描？">
  <Expandable title="答案" defaultOpen="true">
    启动主节点：

    ```bash theme={null}
    osmedeus server --master
    ```

    加入工作节点：

    ```bash theme={null}
    osmedeus worker join --master http://master:8002
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何向分布式池提交任务？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    curl -X POST http://localhost:8002/osm/api/tasks       -H "Authorization: Bearer $TOKEN"       -H "Content-Type: application/json"       -d '{
        "workflow_name": "subdomain-enum",
        "target": "example.com"
      }'
    ```
  </Expandable>
</ResponseField>

***

## 9. 故障排除

<ResponseField name="如何检查工具是否已安装？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    osmedeus install binary --all --check
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何安装缺失的工具？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    # 安装特定工具
    osmedeus install binary --name nuclei --name httpx

    # 安装所有工具
    osmedeus install binary --all
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何查看扫描日志？">
  <Expandable title="答案" defaultOpen="true">
    日志存储在工作区中：

    ```bash theme={null}
    cat ~/osmedeus-base/workspaces/<target>/log/execution.log
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何导出工作区以进行共享？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    osmedeus snapshot export <workspace>
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何导入共享的工作区？">
  <Expandable title="答案" defaultOpen="true">
    ```bash theme={null}
    osmedeus snapshot import snapshot.zip
    ```
  </Expandable>
</ResponseField>

***

## 10. 配置

<ResponseField name="配置文件在哪里？">
  <Expandable title="答案" defaultOpen="true">
    `~/osmedeus-base/osm-settings.yaml`
  </Expandable>
</ResponseField>

<ResponseField name="如何更改默认端口？">
  <Expandable title="答案" defaultOpen="true">
    编辑 `osm-settings.yaml`：

    ```yaml theme={null}
    server:
      port: 9000
    ```

    或者使用 `--port` 标志：

    ```bash theme={null}
    osmedeus server --port 9000
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何配置数据库设置？">
  <Expandable title="答案" defaultOpen="true">
    编辑 `osm-settings.yaml`：

    ```yaml theme={null}
    database:
      db_engine: sqlite3  # 或 postgres
      host: localhost
      port: 5432
      name: osmedeus
      username: user
      password: pass
    ```
  </Expandable>
</ResponseField>

## 11. 清理与卸载

<ResponseField name="如何清理工作区？">
  <Expandable title="答案" defaultOpen="true">
    只需运行以下命令来清理工作区和数据库，并生成默认的 osmedeus 配置：

    ```bash theme={null}
    rm -rf ~/osmedeus-base ~/workspaces-osmedeus
    osmedeus install base --preset
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何清理工作区？">
  <Expandable title="答案" defaultOpen="true">
    只需运行以下命令来清理工作区和数据库：

    ```bash theme={null}
    rm -rf ~/workspaces-osmedeus
    osmedeus db clean --force
    ```
  </Expandable>
</ResponseField>

<ResponseField name="如何卸载 osmedeus？">
  <Expandable title="答案" defaultOpen="true">
    只需运行以下命令：

    ```bash theme={null}
    rm -rf ~/.osmedeus ~/osmedeus-base ~/workspaces-osmedeus
    rm -rf $(which osmedeus)
    ```
  </Expandable>
</ResponseField>

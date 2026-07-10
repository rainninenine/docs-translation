文档API 参考入门Web UI复制页面使用 `vigolium server` 启动 Vigolium 内置的 Web UI，然后可视化扫描数据、检查漏洞发现、选择模块、启动扫描以及调整设置。复制页面运行 `vigolium server` 以启动 Vigolium Web 服务器并打开内置的 Web UI。
vigolium server

默认情况下，服务器监听在 `http://localhost:9002`。在浏览器中打开该 URL 即可使用仪表盘。

对于共享或生产环境，请在启动服务器前设置 `VIGOLIUM_API_KEY`。仅在本地开发且有意禁用身份验证时，才使用 `vigolium server -A`。
# 启动时启用 API 身份验证
export VIGOLIUM_API_KEY=my-secret-key
vigolium server

# 绑定到特定主机和端口
vigolium server --host 127.0.0.1 --service-port 9002

# 仅开发环境：禁用 API 身份验证
vigolium server -A

​在 Web UI 中可以执行的操作
Web UI 为您提供了一个可视化工作区，用于查看 Vigolium 存储在其本地数据库中的数据。当您希望探索结果、调整扫描或跨项目工作而无需停留在终端时，可以使用它。

查看项目级别的扫描摘要和严重性分布。
浏览、搜索和筛选漏洞发现。
打开漏洞发现详情，包含请求、响应、载荷和证据上下文。
查看从 CLI 扫描、API 数据导入或代理数据导入中收集的 HTTP 记录——通过记录来源下拉菜单按来源（扫描、数据导入、代理）筛选，并将任何请求复制为可直接运行的 curl 命令。
查看可用的扫描器模块，包括主动和被动模块。
启动新扫描并监控扫描进度。
通过服务器支持的配置界面修改扫描范围和扫描器配置。

​从仪表盘启动扫描
使用扫描控件针对 URL、导入的流量或已存储在项目中的记录启动后台扫描。Web UI 将请求发送到与 `vigolium scan` 相同的服务器 API，因此扫描输出会落入同一数据库，并在结果写入后立即显示在漏洞发现视图中。
启动扫描时，您可以调整常用选项，例如：

目标 URL 或存储的 HTTP 记录。
扫描策略和强度。
特定模块或模块标签。
扫描范围设置。
主动/被动模块行为。

​与 Burp Suite 插件配合使用
`vigolium server` 也是 Vigolium Burp 插件的本地后端。该插件连接到您正在运行的服务器，通过 `/api/ingest-http` 导入流量，并通过服务器扫描 API 触发扫描。漏洞发现随后会同时出现在 Burp 和 Web UI 中，因为它们存储在同一个 Vigolium 数据库中。
从 Vigolium Burp Suite 插件仓库获取插件源代码和安装文件。

在 Burp 中获得的内容

由 Vigolium 在扫描时实时填充的漏洞发现表。
所选漏洞发现的完整请求/响应对。
扫描流，显示每个模块针对导入请求的运行情况。
上下文菜单操作：发送到 Vigolium、发送到本地扫描、发送到 Agent 扫描。

​配置插件
打开 Vigolium 面板中的“设置”选项卡，将插件指向您的本地服务器。

部分控制内容服务器连接Vigolium API 端点，例如 `http://localhost:9002`，以及来自 `VIGOLIUM_API_KEY` 的 API 密钥。在导入前点击“测试连接”。扫描选项随每个导入请求发送的可选参数，例如超时、扫描 ID 或模块覆盖。留空则使用服务器默认值。键盘快捷键绑定“发送到 Vigolium”、“发送到本地扫描”和“发送到 Agent 扫描”的快捷键。代理拦截将流经 Burp 代理的请求转发到 Vigolium。与“仅限范围内的请求”结合使用，可将导入限制在 Burp 目标范围内。请求统计已转发、成功、失败、排队和正在进行的扫描计数的实时计数器。代理过滤规则按文件扩展名、HTTP 方法或主机允许或拒绝流量，从而避免静态资源和范围外的主机污染数据库。
​Burp 工作流程

使用 API 密钥启动 Vigolium 服务器。
export VIGOLIUM_API_KEY=my-secret-key
vigolium server

在 Burp 中通过“扩展 > 添加”加载插件，打开 Vigolium 选项卡，填写服务器连接信息，然后点击“测试连接”。

通过以下两种方式之一导入流量：

启用代理拦截，将 Burp 代理中范围内的请求转发到 Vigolium。
在代理、重放器或目标中右键单击请求，选择“发送到 Vigolium”。

通过“发送到本地扫描”或“发送到 Agent 扫描”从 Burp 启动扫描。Vigolium 会保留请求头、Cookie、请求体字段、查询参数和路径段作为扫描器输入。

观察漏洞发现流式传输到 Burp 和 Web UI。选择漏洞发现可检查匹配的请求和响应证据。

该插件不会在 Burp 的 JVM 内运行扫描器。它将请求转发到 Vigolium 服务器并轮询服务器以获取漏洞发现，因此扫描使用您的 Vigolium 服务器资源，并且在 Burp 关闭后仍然可用。
有关更低级别的数据导入详情，请参阅服务器数据导入。
​查看可用模块
仪表盘会显示模块注册表，因此您可以在启动扫描前查看 Vigolium 可以测试的内容。使用模块搜索和过滤器按漏洞类别、技术、资源成本或模块类型查找检查项。
为了与 CLI 保持一致，您也可以从终端列出模块：
vigolium module ls
vigolium module ls --tag xss
vigolium module ls --type active

您还可以通过服务器 API 查询模块：
curl -s http://localhost:9002/api/modules \
  -H "Authorization: Bearer my-secret-key" | jq .

有关扫描器模块概念，请参阅模块参考；有关 API 字段，请参阅模块 API。
​调整配置
通过 Web UI 进行的配置更改使用与 CLI 文档中描述的相同的服务器配置模型。当您希望交互式调整扫描行为时，请使用 UI，然后将持久性设置保留在 `~/.vigolium/vigolium-configs.yaml` 中。
常见的配置任务包括：

更新范围规则。
更改并发数、速率限制和每主机限制。
启用或禁用模块组。
调整 CORS 或服务器行为。
查看项目级别的数据隔离。

有关完整的配置详情，请参阅配置和运行服务器。上一页输出与报告Vigolium 支持多种输出格式，用于扫描结果、内容发现数据和爬取输出。本指南介绍了可用的格式、结果结构以及如何查询存储的漏洞发现。下一页⌘IwebsitetwittergithubdiscordxlinkedinPowered by本文档基于 Mintlify 构建和托管，Mintlify 是一个开发者文档平台本页内容在 Web UI 中可以执行的操作从仪表盘启动扫描与 Burp Suite 插件配合使用配置插件Burp 工作流程查看可用模块调整配置
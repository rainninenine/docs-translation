> ## Documentation Index
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# Introduction

> 欢迎阅读 Osmedeus 文档

<Frame>
  <img src="https://mintcdn.com/osmedeus/E-vLGTp3aVA5Tivm/images/introduction/intro.png?fit=max&auto=format&n=E-vLGTp3aVA5Tivm&q=85&s=43a6a6ae5ee5fd29db1cd56af06b62e7" alt="介绍横幅" width="1920" height="1080" data-path="images/introduction/intro.png" />
</Frame>

[Osmedeus](https://github.com/j3ssie/osmedeus) 是一个以安全为核心的声明式编排引擎，它将复杂的工作流自动化简化为可审计的 YAML 定义，并集成了加密数据处理、安全凭据管理和沙箱执行。

无论是初学者还是专家，都能在不牺牲基础设施完整性和安全性的前提下，使用强大且可组合的自动化能力。

## 关键特性

* **声明式 YAML 工作流** - 通过钩子、决策路由、模块排除和条件分支，在多个 Runner（主机、Docker、SSH）上定义管道
* **分布式执行** - 基于 Redis 的主从模式，包含队列系统、Webhook 触发器和跨工作节点的文件同步
* **丰富的函数库** - 80 多个实用函数，包括 nmap 集成、tmux 会话、SSH 执行、TypeScript/Python 脚本、SARIF 解析以及 CDN/WAF 分类
* **事件驱动调度** - 支持 Cron、文件监视和事件触发器，具备过滤、去重和延迟任务队列
* **智能 LLM 步骤** - 工具调用代理循环，包含子代理编排、内存管理和结构化输出
* **云基础设施** - 在 DigitalOcean、AWS、GCP、Linode 和 Azure 上配置并运行扫描，支持成本控制和自动清理
* **丰富的 CLI 界面** - 交互式数据库查询、批量函数评估、工作流 lint、进度条以及全面的使用示例
* **REST API 和 Web UI** - 完整的 API 服务器，支持 Webhook 触发器、数据库查询和嵌入式仪表盘可视化

<Card>
  <img className="block dark:hidden rounded-t-lg w-full" src="https://mintcdn.com/osmedeus/E-vLGTp3aVA5Tivm/images/introduction/hall-of-fame-light.png?fit=max&auto=format&n=E-vLGTp3aVA5Tivm&q=85&s=ba8b1b76ac34683092004f9ce8986abf" alt="浅色模式下的名人堂" width="1281" height="717" data-path="images/introduction/hall-of-fame-light.png" />

  <img className="hidden dark:block rounded-t-lg w-full" src="https://mintcdn.com/osmedeus/E-vLGTp3aVA5Tivm/images/introduction/hall-of-fame-dark.png?fit=max&auto=format&n=E-vLGTp3aVA5Tivm&q=85&s=7227b6cbb7e6a9fc7290dfea70fc51c9" alt="深色模式下的名人堂" width="1920" height="1080" data-path="images/introduction/hall-of-fame-dark.png" />
</Card>

## 快速入门

<Columns cols={2}>
  <Tile href="/getting-started/cli" title="CLI 界面">
    <img src="https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/cli/cli-run-progress.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=a0c448d97d10f955df1589deb2251cc6" alt="CLI 运行预览" width="2704" height="1438" data-path="images/cli/cli-run-progress.png" />
  </Tile>

  <Tile href="/getting-started/web-ui" title="Web UI 界面">
    <img src="https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-assets.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=0ce499f1662852c2a17527d2b49a01b7" alt="Web UI 资产预览" width="2974" height="2378" data-path="images/web-ui/web-ui-assets.png" />
  </Tile>
</Columns>

<Card title="快速开始" href="/quickstart" icon="rocket" horizontal>
  立即上手，在几分钟内运行您的第一个 Osmedeus 工作流。
</Card>

***

## 高级安装与配置

<Columns cols={2}>
  <Card title="安装" href="/getting-started/installation" icon="download">
    在不同平台上安装 Osmedeus 的详细说明。
  </Card>

  <Card title="配置" href="/getting-started/configuration" icon="gear">
    配置引擎、Runner 和环境变量。
  </Card>

  <Card title="部署" href="/getting-started/deployment" icon="server">
    在分布式环境或生产环境中部署 Osmedeus。
  </Card>

  <Card title="开发" href="/getting-started/development" icon="display-code">
    为贡献或扩展 Osmedeus 的开发者提供的资源。
  </Card>
</Columns>

## 理解 Osmedeus

### 核心概念

| 页面                                     | 描述                                                                         |
| ---------------------------------------- | ----------------------------------------------------------------------------------- |
| [体系结构](concepts/architecture.md) | 分层架构与数据流                                                  |
| [工作流](concepts/workflows.md)       | 模块与 Flow 对比，执行生命周期                                                 |
| [模板](concepts/templates.md)       | 变量插值与内置变量                                       |
| [Runner](concepts/runners.md)           | 主机、Docker、SSH 执行环境                                            |
| [函数](concepts/functions.md)       | JavaScript 实用函数，绑定到核心引擎，用于工作流步骤 |

### 高级主题

| 页面                                             | 描述                          |
| ------------------------------------------------ | ------------------------------------ |
| [分布式执行](advanced/distributed.md) | 主从架构           |
| [调度](advanced/scheduling.md)             | Cron、事件和文件监视触发器 |
| [LLM 集成](advanced/llm.md)               | AI 驱动的工作流步骤            |
| [快照](advanced/snapshots.md)               | 项目空间导出与导入          |

***

### 工作流

| 页面                                      | 描述                                |
| ----------------------------------------- | ------------------------------------------ |
| [概述](workflows/overview.md)         | YAML 结构和工作流类型          |
| [步骤类型](workflows/step-types.md)     | 所有 8 种步骤类型及示例             |
| [Flow](workflows/flows.md)               | 模块编排与依赖关系      |
| [变量](workflows/variables.md)       | 参数、导出、变量传播  |
| [控制流](workflows/control-flow.md) | 条件、处理器和决策路由 |

### 扩展 Osmedeus

| 页面                                        | 描述                |
| ------------------------------------------- | -------------------------- |
| [步骤类型](extending/step-types.md)       | 添加自定义步骤执行器  |
| [Runner](extending/runners.md)             | 实现新的 Runner 类型 |
| [函数](extending/functions.md)         | 注册实用函数 |
| [CLI 命令](extending/cli-commands.md)   | 添加新的 CLI 命令       |
| [API 端点](extending/api-endpoints.md) | 添加新的 REST 端点     |

## 参考

| 页面                                            | 描述          |
| ----------------------------------------------- | -------------------- |
| [工作流模式](reference/workflow-schema.md) | 完整的 YAML 模式 |
| [变量](reference/variables.md)             | 内置变量   |
| [类型](reference/types.md)                     | Go 类型定义  |

## 完整功能列表

* **声明式 YAML 工作流** - 使用简单、可读的 YAML 语法定义侦察管道
* **多种 Runner** - 在本地主机、Docker 容器或通过 SSH 的远程机器上执行
* **事件驱动触发器** - Cron 调度、文件监视和基于事件的工作流触发器，支持去重和过滤函数
* **模板引擎** - 强大的变量插值，支持内置和自定义变量
* **实用函数** - 丰富的函数库，包含事件生成、批量处理和 JSON 操作
* **REST API 服务器** - 以编程方式管理、触发和取消工作流
* **分布式执行** - 基于 Redis 的主从模式进行并行扫描（工作节点标识为 `wosm-<uuid8>`）
* **通知** - Telegram 机器人和 Webhook 集成
* **云存储** - 兼容 S3 的存储，用于产物管理
* **LLM 集成** - AI 驱动的工作流步骤，支持聊天补全、嵌入和代理工具调用循环
* **代理步骤类型** - 代理型 LLM 执行，支持工具调用、子代理和内存管理
* **SAST 集成** - 解析 Semgrep、Trivy、Kingfisher、Bearer 的 SARIF 输出，支持数据库导入和 Markdown 报告
* **语言检测** - 自动检测源代码仓库的主要编程语言（支持 26 种以上语言）
* **预设安装** - 从精选预设仓库进行可复现的部署
* **工作流钩子** - 通过 `hooks` 字段在扫描前后执行设置和清理步骤
* **队列系统** - 延迟任务执行，支持数据库和 Redis 轮询，可配置并发
* **Nmap 集成** - 端口扫描，自动将 XML/gnmap 转换为 JSONL 并导入数据库
* **Tmux 会话** - 通过 tmux 管理后台进程（创建、捕获、发送、终止会话）
* **SSH 与同步** - 跨分布式工作节点的远程执行和文件同步
* **TypeScript 执行** - 通过 Bun 运行时运行内联 TypeScript 或 TS 文件
* **Webhook 触发器** - 通过无需身份验证的 Webhook URL 触发工作流运行
* **CDN/WAF 分类** - 根据 httpx 数据自动分类资产（CDN、云、WAF）
* **模块排除** - 通过精确名称或模糊子串匹配从 Flow 中排除模块
* **云基础设施** - 跨多个供应商配置和管理云实例
> ## 文档索引
> 获取完整文档索引，请访问：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# Web UI

> Osmedeus 的服务器与 Web 界面

本文档介绍 Osmedeus 的 Web UI，它是运行工作流、模块及其他实用工具的主要入口。

!!! note "Osmedeus 服务器也充当事件接收器，处理来自其他运行的事件并管理定时扫描。详情请参阅[事件驱动](/advanced/event-driven)。"

通过运行以下命令启动 Web UI：

```bash
osmedeus serve
```

然后在浏览器中打开 `https://localhost:8002`。

## 如何登录 Web UI

![web-ui-login](https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-login.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=dfba46c1700d447635d4cad73be37d81)

!!! tip "您的默认密码位于 `$HOME/osmedeus-base/osm-settings.yaml` 中"

您可以通过运行以下命令查看默认凭据（这些凭据是自动生成的）：

```bash
osmedeus config view server.username
osmedeus config view server.password
```

!!! danger "许多 API 端点被有意设计为在您的机器上执行代码，因此请确保您的 Web UI 和 API 端点使用强凭据进行保护。详情请参阅[安全警告](/others/security-warning)。"

您还可以通过运行以下命令更改默认密码并设置 API 密钥认证：

```bash
osmedeus config set server.username "osmedeus"
osmedeus config set server.password "$(openssl rand -hex 12)"
# api key auth requires a jwt secret signing key
osmedeus config set server.jwt.secret_signing_key "$(openssl rand -hex 32)"
osmedeus config set server.enabled_auth_api true
osmedeus config set server.auth_api_key "$(openssl rand -hex 24)"

## API Key Authentication Settings
# server:
#   enabled_auth_api: true
#   auth_api_key: "your-secure-api-key"

```

***

## 所有 Web UI 页面

Web UI 包含以下页面，允许您查看和管理资产、工作空间、漏洞及其他实用工具。

### 1. 资产与工作空间

这是 Web UI 的主页面，您可以在此查看和管理资产、工作空间、漏洞以及工作流生成的工件。

![web-ui-workspace](https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-workspace.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=0b2294cbd78b0a220b0f660a3239b845)


  ![web-ui-assets](https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-assets.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=0ce499f1662852c2a17527d2b49a01b7)

  ![web-ui-vuln](https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-vuln.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=03e1ce746624549ce5c83b42e1810139)



  ![web-ui-assets](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/web-ui/web-ui-assets-light.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=ad0fe412789e8619e886d487843c7d7e)

  ![web-ui-vuln](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/web-ui/web-ui-vuln-light.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=1673c91a94e46376fbd29bf092b14246)


您还可以选择增强 UI，以更清晰、语法高亮的布局显示工件——非常适合查看 Markdown 或 HTML 报告。


  ![web-ui-artifact-list](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/web-ui/web-ui-artifact-list.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=8c503eba5dd87d53f1f87babddb0f719)

  ![web-ui-artifact-details](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/web-ui/web-ui-artifact-details.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=b245a40f6f8f0670ed2394331ce106ca)


### 2. 开始新扫描（简单运行与定时扫描）

您可以在此通过选择工作流和目标资产，并设置所有额外参数和调度选项来开始新扫描。

![web-ui-new](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/web-ui/web-ui-new-scan.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=6c602f66a6050a88b06d6544dd17cedc)

开始新扫描后，您可以在扫描列表中查看进度和结果。

![web-ui-list-scan](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/web-ui/web-ui-list-scan.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=72cdee67fb77700a2f7752728aec5480)

### 3. 设置与安装注册表

您可以在此配置设置并安装 Osmedeus 的注册表。


  ![web-ui-install-registry](https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-install-registry.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=abdb5e6a931f20ea0602658ccd8d9fbf)

  ![web-ui-settings](https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-settings.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=6ec05384cb97b44c35e379d423fc3cf3)


### 4. 实用工具函数与调度

您可以在此安排工作流在特定时间或间隔运行，还可以使用 LLM 聊天获取工作流方面的帮助。


  ![web-ui-schedule](https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-schedule.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=557450eae49d2c08d54cd78b1bd4e4e7)

  ![web-ui-llm-chat](https://mintcdn.com/osmedeus/v2F2CQFtcKul_NUM/images/web-ui/web-ui-llm-chat.png?fit=max&auto=format&n=v2F2CQFtcKul_NUM&q=85&s=d13704496b57f5c8047539aca509b5a0)

  ![web-ui-utility-functions](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/web-ui/web-ui-utility-functions.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=f0fdc7f162eccf5205e511611cc5f928)


## 工作流可视化与编辑器

通过 xyflow 对工作流进行美观的可视化展示，并在 Web UI 中提供编辑器。

![web-ui-workflow1](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/workflow-editor/web-ui-workflow-1.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=d1b131606a725342236d09695d994b96)

![web-ui-workflow2](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/workflow-editor/web-ui-workflow-2.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=39238f673302ed02c24bcd98739b0cfd)

![web-ui-workflow3](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/workflow-editor/web-ui-workflow-3.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=5b64d52474163fe14c1bfda3daec75e5)

![web-ui-workflow4](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/workflow-editor/web-ui-workflow-4.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=f6805dec64cf3711338d18f3f8f35b2f)

![web-ui-workflow4](https://mintcdn.com/osmedeus/rQ49bCQbs9dKARCy/images/workflow-editor/web-ui-workflow-light.png?fit=max&auto=format&n=rQ49bCQbs9dKARCy&q=85&s=c0cbec3d518dce7702d2068794d9df0d)

## 与 Web UI 通信

登录后，您可以使用 Web UI 运行工作流、查看结果以及管理您的 Osmedeus 安装。

```bash Request theme={null}
curl --request POST   --url https://localhost:8002/osm/api/runs   --header 'Authorization: Bearer $TOKEN'   --header 'Content-Type: application/json'   --data '{"flow": "general", "target": "example.com", "distributed": true}'
```

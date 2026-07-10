> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 通知与CDN配置

> 本指南涵盖高级配置选项，包括通知和CDN设置

通知和CDN设置可直接在设置文件中修改。本页提供通过CLI进行最常见配置的快速参考，以及如何验证您的设置。

## 通知配置

```bash
## Telegram 设置
osmedeus config set notification.enabled true
osmedeus config set notification.telegram.enabled true
osmedeus config set notification.provider telegram
osmedeus config set notification.telegram.bot_token "12345:your-token"
osmedeus config set notification.telegram.chat_id "-1001234567890"
```

通过使用工具函数 `osmedeus eval 'notify_telegram("**hola** osmedeus from cli")'` 发送示例消息来验证您的设置。

![通知函数](https://mintcdn.com/osmedeus/8zvZUJvRBGDXZt0n/images/noti/noti-funcs.png?fit=max&auto=format&n=8zvZUJvRBGDXZt0n&q=85&s=5c7ea0a243ff111ce3a014986c2c870e)

使用 `osmedeus func ls noti` 列出所有可用的通知函数。

### 如何获取您的Telegram令牌和频道ID

以下是获取Telegram令牌和频道ID的快速指南：

```bash
搜索 @BotFather 并创建您的机器人，然后获取令牌

# 获取频道ID
curl "https://api.telegram.org/bot$TELEGRAM_TOKEN/getUpdates" | jq 

# 发送测试消息
curl -X POST "https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage" -d chat_id=-<channel-id> -d text="Hello, this is a broadcast to the channel"
```

## CDN配置

使用 `osmedeus func ls cdn` 列出所有可用的CDN函数。以下是我常用的一些函数：

![CDN函数](https://mintcdn.com/osmedeus/8zvZUJvRBGDXZt0n/images/noti/cdn-funcs.png?fit=max&auto=format&n=8zvZUJvRBGDXZt0n&q=85&s=a3bbbe7a0fac73fa97bca48205e12f0f)

```bash

osmedeus eval 'cdn_ls_tree()'

# 下载和上传结果
osmedeus eval cdn_download("workspace_result.jsonl", "workspace_result.jsonl")
osmedeus eval cdn_upload("workspace_result.jsonl", "workspace_result.jsonl")

# 同步文件夹到远程
osmedeus eval 'cdn_sync_upload("local", "remote")'

osmedeus eval 'cdn_ls_tree("targets/")'
```

### Cloudflare R2

```bash
## R2的CDN设置
osmedeus config set storage.provider 'r2'
osmedeus config set storage.access_key_id '<r2-key-id>'
osmedeus config set storage.bucket '<r2-bucket-name>'
osmedeus config set storage.secret_access_key '<r2-access-key>'
osmedeus config set storage.endpoint '<endpoint>.r2.cloudflarestorage.com'
osmedeus config set storage.presign_expiry '2h'
osmedeus config set storage.use_ssl 'true'
osmedeus config set storage.enabled 'true'
```

通过使用工具函数 `osmedeus eval 'cdn_ls_tree()'` 发送示例消息来验证您的设置。

### Google Cloud Storage

```bash
## GCS的CDN设置
osmedeus config set storage.provider 'gcs'
osmedeus config set storage.access_key_id '<gcs-key-id>'
osmedeus config set storage.secret_access_key '<gcs-access-key>'
osmedeus config set storage.bucket '<gcs-bucket-name>'
osmedeus config set storage.endpoint 'storage.googleapis.com'
osmedeus config set storage.region 'us-east-1'
osmedeus config set storage.use_ssl 'true'
osmedeus config set storage.presign_expiry '1h'
osmedeus config set storage.enabled 'true'
```

通过使用工具函数 `osmedeus eval 'cdn_ls_tree()'` 发送示例消息来验证您的设置。

## 完整通知与CDN配置

<details>
<summary>完整通知与CDN配置</summary>

  ```yaml
  # =============================================================================
  # 通知配置
  # =============================================================================
  # 扫描完成或发现有趣结果时发送通知
  notification:
    # 通知供应商："telegram"（未来支持：slack, discord, webhook）
    provider: telegram

    # 主开关，启用/禁用所有通知
    enabled: false

    # Telegram机器人设置
    # 通过 @BotFather 创建机器人并获取令牌
    # 通过向 @userinfobot 发送消息获取您的聊天ID
    telegram:
      # 来自 @BotFather 的机器人令牌
      bot_token: ""

      # 接收消息的聊天ID（可以是用户或群组）
      chat_id: 0

      # 启用Telegram通知
      enabled: false

      # 按名称发送到特定频道/群组的频道映射
      # 使用 notify_telegram_channel("#name", "message") 或 send_telegram_file_channel("#name", "/path/to/file")
      # 也可以直接使用数字聊天ID：notify_telegram_channel("-1001234567890", "message")
      telegram_channel_map:
        # alerts: -1001234567890
        # reports: -1009876543210

    # Webhook设置
    # 配置多个Webhook端点用于通知
    # Webhook在工作流事件（完成、失败等）时触发
    webhooks:
      # 示例Webhook配置
      - url: "https://example.com/webhook"
        # 启用/禁用此Webhook
        enabled: false
        # 自定义HTTP头（例如，用于身份验证）
        headers:
          Authorization: "Bearer your-token-here"
          Content-Type: "application/json"
        # 请求超时时间（秒，默认：30）
        timeout: 30
        # 失败时的重试次数（默认：3）
        retry_count: 3
        # 跳过TLS证书验证（默认：false）
        # 对于自签名证书设置为true
        skip_tls_verify: false
        # 事件过滤器 - 仅在这些事件时触发（空 = 所有事件）
        # 可用事件：workflow_completed, workflow_failed, workflow_cancelled
        events:
          - workflow_completed
          - workflow_failed

      # 第二个Webhook示例（例如，Slack incoming webhook）
      # - url: "https://hooks.slack.com/services/xxx/yyy/zzz"
      #   enabled: false
      #   headers: {}
      #   events:
      #     - workflow_completed

  # =============================================================================
  # 云存储配置（可选）
  # =============================================================================
  # 用于备份扫描结果的S3兼容存储
  # 支持AWS S3, MinIO, Cloudflare R2, Google Cloud Storage, DigitalOcean Spaces, Oracle OCI
  storage:
    # 存储供应商："s3", "minio", "r2", "gcs", "spaces", "oci"
    # 供应商决定端点解析和默认设置
    provider: s3

    # 存储端点URL（大多数供应商自动解析）
    # 留空以根据供应商和区域/账户ID自动解析
    # 显式示例：
    #   AWS S3: "s3.us-east-1.amazonaws.com"
    #   MinIO: "localhost:9000"
    #   R2: 将从account_id自动解析
    #   GCS: "storage.googleapis.com"
    #   Spaces: 将从区域自动解析（例如，"nyc3.digitaloceanspaces.com"）
    #   OCI: 将从account_id（命名空间）和区域自动解析
    endpoint: ""

    # 访问凭证
    access_key_id: ""
    secret_access_key: ""

    # 用于存储结果的存储桶名称
    bucket: ""

    # 云区域（例如，us-east-1, eu-west-1）
    # 必需：s3, spaces, oci
    region: us-east-1

    # 账户ID或命名空间（供应商特定）
    # R2: 您的Cloudflare账户ID
    # OCI: 您的对象存储命名空间
    account_id: ""

    # 使用SSL/TLS连接（默认：云供应商为true）
    use_ssl: true

    # 强制路径样式URL（按供应商自动配置）
    # 对于MinIO, R2, OCI设置为true；对于S3, GCS, Spaces设置为false
    path_style: false

    # 默认预签名URL过期时间（例如，"1h", "30m", "24h"）
    # 当未指定过期时间时，由cdnGetPresignedURL使用
    presign_expiry: "1h"

    # 启用云存储上传
    enabled: false


  ```


</details>
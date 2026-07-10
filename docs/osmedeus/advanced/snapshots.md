> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 快照（Snapshots）

将项目空间（Workspace）导出和导入为压缩的 ZIP 归档文件。

## 概述

快照支持：

* 备份和恢复项目空间
* 共享扫描结果
* 在机器间迁移
* 归档已完成的评估

## 导出

### CLI

```bash
# 导出项目空间
osmedeus snapshot export example.com

# 自定义输出路径
osmedeus snapshot export example.com -o /backups/example.zip

# 导出到指定目录
osmedeus snapshot export example.com -o ~/archives/
```

### API

```bash
curl -X POST http://localhost:8002/osm/api/snapshots/export \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"workspace": "example.com"}'
```

### 内容

导出的 ZIP 包含：

```
example.com-20240115-143022.zip
├── workspace/                    # 所有项目空间文件
│   ├── subdomain-enum/
│   ├── http-probe/
│   └── ...
├── metadata.json                 # 导出元数据
└── database.json                 # 数据库记录
    ├── assets
    ├── vulnerabilities
    └── runs
```

## 导入

### CLI

```bash
# 从本地文件导入
osmedeus snapshot import ~/backup.zip

# 从 URL 导入
osmedeus snapshot import https://example.com/workspace.zip

# 强制覆盖已存在的项目空间
osmedeus snapshot import ~/backup.zip --force

# 仅导入文件（跳过数据库）
osmedeus snapshot import ~/backup.zip --skip-db
```

### API

```bash
# 从 URL 导入
curl -X POST http://localhost:8002/osm/api/snapshots/import \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"source": "https://example.com/backup.zip"}'

# 带选项导入
curl -X POST http://localhost:8002/osm/api/snapshots/import \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/path/to/backup.zip",
    "force": true,
    "skip_db": false
  }'
```

## 列出快照

### CLI

```bash
osmedeus snapshot list
```

### API

```bash
curl http://localhost:8002/osm/api/snapshots \
  -H "Authorization: Bearer $TOKEN"
```

### 输出

```
可用快照：

┌────────────────────────────────────────────┬────────────┬─────────────────────┐
│ 文件名                                     │ 大小       │ 创建时间            │
├────────────────────────────────────────────┼────────────┼─────────────────────┤
│ example.com-20240115-143022.zip            │ 12.5 MB    │ 2024-01-15 14:30:22 │
│ target.org-20240114-091500.zip             │ 8.2 MB     │ 2024-01-14 09:15:00 │
└────────────────────────────────────────────┴────────────┴─────────────────────┘
```

## 下载

```bash
# 通过 API 下载
curl http://localhost:8002/osm/api/snapshots/download/example.com-backup.zip \
  -H "Authorization: Bearer $TOKEN" \
  -o local-backup.zip
```

## 删除

```bash
curl -X DELETE http://localhost:8002/osm/api/snapshots/example.com-backup.zip \
  -H "Authorization: Bearer $TOKEN"
```

## 存储位置

默认：`~/osmedeus-base/snapshots/`

在 `osm-settings.yaml` 中配置：

```yaml
environments:
  storages: "{{base_folder}}/snapshots"
```

## 使用场景

### 在破坏性扫描前备份

```bash
# 备份当前状态
osmedeus snapshot export example.com

# 运行可能具有破坏性的扫描
osmedeus run -f aggressive-scan -t example.com

# 必要时恢复
osmedeus snapshot import ~/osmedeus-base/snapshots/example.com-*.zip --force
```

### 共享结果

```bash
# 导出项目空间
osmedeus snapshot export client-assessment -o /tmp/results.zip

# 通过云存储共享
aws s3 cp /tmp/results.zip s3://bucket/results.zip
```

### 迁移到另一台机器

```bash
# 在源机器上
osmedeus snapshot export example.com -o /tmp/workspace.zip
scp /tmp/workspace.zip user@target:/tmp/

# 在目标机器上
osmedeus snapshot import /tmp/workspace.zip
```

### 归档已完成的项目

```bash
# 导出并压缩
osmedeus snapshot export project-2024 -o /archives/project-2024.zip

# 移除项目空间
rm -rf ~/osmedeus-base/workspaces/project-2024

# 后续需要时恢复
osmedeus snapshot import /archives/project-2024.zip
```

## 元数据

每个快照包含 `metadata.json`：

```json
{
  "version": "1.0",
  "workspace": "example.com",
  "created_at": "2024-01-15T14:30:22Z",
  "osmedeus_version": "2.0.0",
  "files_count": 156,
  "total_size": 13107200,
  "records": {
    "assets": 423,
    "vulnerabilities": 15,
    "runs": 3
  }
}
```

## 数据库记录

`database.json` 包含：

```json
{
  "assets": [
    {
      "id": 1,
      "workspace": "example.com",
      "asset_value": "sub.example.com",
      "url": "https://sub.example.com",
      "status_code": 200,
      ...
    }
  ],
  "vulnerabilities": [
    {
      "id": 1,
      "workspace": "example.com",
      "title": "SQL Injection",
      "severity": "high",
      ...
    }
  ],
  "runs": [
    {
      "id": "run-abc123",
      "workflow_name": "full-recon",
      "target": "example.com",
      ...
    }
  ]
}
```

## 导入选项

### 强制覆盖

```bash
osmedeus snapshot import backup.zip --force
```

* 删除现有项目空间
* 移除现有数据库记录
* 从快照中全新导入

### 跳过数据库

```bash
osmedeus snapshot import backup.zip --skip-db
```

* 仅导入文件
* 保留现有数据库记录
* 适用于仅文件更新

## 云存储

### 导出到 S3

```bash
# 本地导出
osmedeus snapshot export example.com -o /tmp/backup.zip

# 上传到 S3
aws s3 cp /tmp/backup.zip s3://bucket/snapshots/

# 或在设置中配置存储
storage:
  enabled: true
  provider: s3
  bucket: osmedeus-snapshots
```

### 从 S3 导入

```bash
# 生成预签名 URL
URL=$(aws s3 presign s3://bucket/snapshots/backup.zip)

# 从 URL 导入
osmedeus snapshot import "$URL"
```

## 最佳实践

1. **定期备份** - 安排快照导出
2. **导入后验证** - 检查文件完整性
3. **使用描述性名称** - 包含日期和用途
4. **安全存储** - 加密敏感快照
5. **清理旧快照** - 管理存储空间

## 故障排除

### 导入失败

```bash
# 检查文件完整性
unzip -t backup.zip

# 尝试使用详细输出
osmedeus snapshot import backup.zip -v
```

### 导入后文件缺失

```bash
# 检查是否使用了 --skip-db
# 重新导入并包含数据库
osmedeus snapshot import backup.zip
```

### 快照体积过大

```bash
# 检查项目空间内容
du -sh ~/osmedeus-base/workspaces/example.com/*

# 导出前清理不必要的文件
rm -rf ~/osmedeus-base/workspaces/example.com/temp/
```

## 下一步

* [快照 CLI](../cli/snapshot.md) - CLI 参考
* [API 概述](../api/overview.md) - 快照端点
* [项目空间](../concepts/workflows.md) - 项目空间结构
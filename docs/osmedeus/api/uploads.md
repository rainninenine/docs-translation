---
title: "文件上传"
description: "上传目标文件和工作流"
---

# 文件上传

## 上传输入文件

上传包含输入列表（目标、URL 等）的文件，供后续运行使用。

```bash
curl -X POST http://localhost:8002/osm/api/upload-file \
  -H "Authorization: Bearer ***" \
  -F "file=@targets.txt"
```

**响应：**
```json
{
  "message": "File uploaded",
  "filename": "1704326400000000000_targets.txt",
  "path": "/home/user/osmedeus-base/data/uploads/1704326400000000000_targets.txt",
  "size": 1024,
  "lines": 50
}
```

返回的 `path` 可以在后续运行请求中用作目标。

---

## 上传工作流

上传原始 YAML 工作流文件并保存到工作流目录。

```bash
curl -X POST http://localhost:8002/osm/api/workflow-upload \
  -H "Authorization: Bearer ***" \
  -F "file=@my-custom-workflow.yaml"
```

**响应：**
```json
{
  "message": "Workflow uploaded",
  "name": "my-custom-workflow",
  "kind": "module",
  "description": "A custom security workflow",
  "path": "/home/user/osmedeus-base/workflows/modules/my-custom-workflow.yaml"
}
```

工作流文件必须是有效的 YAML，扩展名为 `.yaml` 或 `.yml`。将根据工作流类型保存到 `flows/` 或 `modules/` 子目录。

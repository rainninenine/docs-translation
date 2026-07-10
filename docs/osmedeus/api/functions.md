---
title: "函数"
description: "执行和列出工具函数"
---

# 函数

## 执行工具函数

执行一个带有模板渲染和 JavaScript 执行的工具函数脚本。

```bash
curl -X POST http://localhost:8002/osm/api/functions/eval \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "script": "trim(\"  hello  \")"
  }'
```

**带目标变量：**
```bash
curl -X POST http://localhost:8002/osm/api/functions/eval \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "script": "fileExists(\"{{target}}\")",
    "target": "/tmp/test.txt"
  }'
```

**带自定义参数：**
```bash
curl -X POST http://localhost:8002/osm/api/functions/eval \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "script": "log_info(\"{{host}}:{{port}}\")",
    "params": {
      "host": "localhost",
      "port": "8080"
    }
  }'
```

**请求体：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `script` | string | 是 | 要执行的 JavaScript 脚本 |
| `target` | string | 否 | `{{target}}` 变量的目标值 |
| `params` | object | 否 | 用于模板渲染的附加参数 |

**响应：**
```json
{
  "result": "hello",
  "rendered_script": "trim(\"  hello  \")"
}
```

---

## 列出工具函数

获取所有可用工具函数的分类列表。

```bash
curl http://localhost:8002/osm/api/functions/list \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "functions": {
    "file": [
      {"name": "fileExists(path)", "description": "Check if file exists", "return_type": "bool"},
      {"name": "fileLength(path)", "description": "Count non-empty lines in file", "return_type": "int"},
      {"name": "readFile(path)", "description": "Read entire file contents", "return_type": "string"},
      {"name": "writeFile(path, content)", "description": "Write content to file", "return_type": "bool"},
      {"name": "appendFile(path, content)", "description": "Append content to file", "return_type": "bool"},
      {"name": "removeFile(path)", "description": "Delete a file", "return_type": "bool"},
      {"name": "copyFile(src, dst)", "description": "Copy file to destination", "return_type": "bool"},
      {"name": "mergeFiles(pattern, output)", "description": "Merge multiple files matching pattern", "return_type": "bool"}
    ],
    "string": [
      {"name": "trim(str)", "description": "Trim whitespace", "return_type": "string"},
      {"name": "split(str, delim)", "description": "Split string by delimiter", "return_type": "[]string"},
      {"name": "replace(str, old, new)", "description": "Replace all occurrences", "return_type": "string"},
      {"name": "contains(str, substr)", "description": "Check if string contains substring", "return_type": "bool"},
      {"name": "toLowerCase(str)", "description": "Convert string to lowercase", "return_type": "string"},
      {"name": "toUpperCase(str)", "description": "Convert string to uppercase", "return_type": "string"},
      {"name": "join(array, delim)", "description": "Join array elements with delimiter", "return_type": "string"}
    ],
    "utility": [
      {"name": "len(val)", "description": "Get length of string or array", "return_type": "int"},
      {"name": "exec_cmd(command)", "description": "Execute bash command and return output", "return_type": "string"},
      {"name": "isEmpty(val)", "description": "Check if value is empty/nil", "return_type": "bool"},
      {"name": "commandExists(name)", "description": "Check if command is installed", "return_type": "bool"},
      {"name": "sleep(seconds)", "description": "Sleep for specified seconds", "return_type": "void"}
    ],
    "http": [
      {"name": "http_get(url)", "description": "Make HTTP GET request", "return_type": "object"},
      {"name": "http_post(url, body)", "description": "Make HTTP POST request with JSON body", "return_type": "object"},
      {"name": "httpRequest(url, method, headers, body)", "description": "Make HTTP request with full control", "return_type": "object"}
    ],
    "logging": [
      {"name": "log_info(message)", "description": "Log info message with [INFO] prefix", "return_type": "void"},
      {"name": "log_debug(message)", "description": "Log debug message with [DEBUG] prefix", "return_type": "void"},
      {"name": "log_warn(message)", "description": "Log warning message with [WARN] prefix", "return_type": "void"},
      {"name": "log_error(message)", "description": "Log error message with [ERROR] prefix", "return_type": "void"}
    ],
    "generation": [
      {"name": "randomString(length)", "description": "Generate random alphanumeric string", "return_type": "string"},
      {"name": "uuid()", "description": "Generate UUID v4", "return_type": "string"},
      {"name": "timestamp()", "description": "Get current Unix timestamp", "return_type": "int"}
    ],
    "encoding": [
      {"name": "base64Encode(str)", "description": "Encode string to base64", "return_type": "string"},
      {"name": "base64Decode(str)", "description": "Decode base64 string", "return_type": "string"},
      {"name": "urlEncode(str)", "description": "URL encode string", "return_type": "string"},
      {"name": "urlDecode(str)", "description": "URL decode string", "return_type": "string"}
    ],
    "unix_commands": [
      {"name": "sortUnix(inputFile, outputFile)", "description": "Sort file and remove duplicates", "return_type": "bool"},
      {"name": "diff_unix(file1, file2, outputFile)", "description": "Get difference between two files", "return_type": "bool"},
      {"name": "gitClone(url, destPath)", "description": "Clone git repository", "return_type": "bool"},
      {"name": "gitPull(repoPath)", "description": "Pull latest changes from git remote", "return_type": "bool"}
    ],
    "database": [
      {"name": "db_insert_asset(workspace, data)", "description": "Insert asset into database", "return_type": "bool"},
      {"name": "db_query_assets(workspace, filter)", "description": "Query assets from database", "return_type": "[]object"},
      {"name": "db_update_workspace_stats(workspace)", "description": "Update workspace statistics", "return_type": "bool"}
    ]
  }
}
```

**可用函数类别：**
- `file` - 文件操作（fileExists、readFile、removeFile 等）
- `string` - 字符串处理（trim、split、replace 等）
- `type_conversion` - 类型转换（parseInt、toString 等）
- `utility` - 通用工具（len、isEmpty、exec_cmd）
- `logging` - 日志函数（log_info、log_debug）
- `http` - HTTP 请求
- `generation` - 随机值（randomString、uuid）
- `encoding` - Base64 编解码
- `notification` - Telegram 通知
- `cdn_storage` - 云存储操作
- `unix_commands` - Unix 命令封装（sortUnix、gitClone 等）
- `archive` - 归档操作（zip_dir、unzip_dir）
- `markdown` - Markdown 渲染函数
- `database` - 数据库操作

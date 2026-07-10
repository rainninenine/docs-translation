---
title: "安装"
description: "从注册表安装二进制文件和工作流"
---

# 安装

## 获取注册表信息

获取带有安装状态的二进制注册表元数据。支持两种模式：
- `direct-fetch`（默认）：从注册表 JSON 获取二进制下载 URL
- `nix-build`：按类别分组的 Nix flake 二进制文件

**查询参数：**

| 参数 | 类型 | 默认值 | 描述 |
|-----------|------|---------|-------------|
| `registry_mode` | string | `direct-fetch` | 注册表模式：`direct-fetch` 或 `nix-build` |

---

### 直接获取模式（默认）

返回每个平台/架构的二进制元数据及下载 URL。

```bash
curl http://localhost:8002/osm/api/registry-info \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "registry_mode": "direct-fetch",
  "registry_url": "https://raw.githubusercontent.com/osmedeus/osmedeus-base/main/registry-metadata.json",
  "binaries": {
    "nuclei": {
      "desc": "Vulnerability scanner",
      "tags": ["vuln", "scanner"],
      "version": "3.0.0",
      "linux": {
        "amd64": "https://github.com/projectdiscovery/nuclei/releases/download/v3.0.0/nuclei_3.0.0_linux_amd64.zip",
        "arm64": "https://github.com/projectdiscovery/nuclei/releases/download/v3.0.0/nuclei_3.0.0_linux_arm64.zip"
      },
      "darwin": {
        "amd64": "https://github.com/projectdiscovery/nuclei/releases/download/v3.0.0/nuclei_3.0.0_darwin_amd64.zip",
        "arm64": "https://github.com/projectdiscovery/nuclei/releases/download/v3.0.0/nuclei_3.0.0_darwin_arm64.zip"
      },
      "installed": true,
      "path": "/usr/local/bin/nuclei"
    },
    "amass": {
      "desc": "In-depth attack surface mapping",
      "tags": ["recon", "subdomain"],
      "version": "4.0.0",
      "linux": {
        "amd64": "https://github.com/owasp-amass/amass/releases/download/v4.0.0/amass_linux_amd64.zip"
      },
      "darwin": {
        "amd64": "https://github.com/owasp-amass/amass/releases/download/v4.0.0/amass_darwin_amd64.zip"
      },
      "installed": false,
      "path": ""
    }
  }
}
```

**响应字段（direct-fetch）：**

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `registry_mode` | string | 始终为 `"direct-fetch"` |
| `registry_url` | string | 二进制注册表来源的 URL |
| `binaries` | object | 二进制名称到其元数据的映射 |
| `binaries[name].desc` | string | 二进制工具的说明 |
| `binaries[name].tags` | []string | 二进制文件的标签/分类 |
| `binaries[name].version` | string | 二进制文件的版本 |
| `binaries[name].linux` | object | 按架构（amd64、arm64）的 Linux 下载 URL |
| `binaries[name].darwin` | object | 按架构的 macOS 下载 URL |
| `binaries[name].windows` | object | 按架构的 Windows 下载 URL |
| `binaries[name].command-linux` | object | 按架构的 Linux 安装命令 |
| `binaries[name].command-darwin` | object | 按架构的 macOS 安装命令 |
| `binaries[name].installed` | boolean | 二进制文件当前是否已安装 |
| `binaries[name].path` | string | 已安装二进制文件的完整路径 |

---

### Nix 构建模式

返回按类别分组的 Nix flake 二进制文件及注册表元数据。

```bash
curl "http://localhost:8002/osm/api/registry-info?registry_mode=nix-build" \
  -H "Authorization: Bearer ***"
```

**响应：**
```json
{
  "registry_mode": "nix-build",
  "nix_installed": true,
  "categories": [
    {
      "name": "Subdomain",
      "tools": [
        {
          "name": "amass",
          "desc": "In-depth attack surface mapping and asset discovery",
          "tags": ["recon", "subdomain"],
          "version": "4.2.0",
          "repo_link": "https://github.com/owasp-amass/amass",
          "installed": true,
          "path": "/home/user/.nix-profile/bin/amass"
        },
        {
          "name": "subfinder",
          "desc": "Fast passive subdomain enumeration tool",
          "tags": ["recon", "subdomain"],
          "version": "2.6.0",
          "installed": false
        }
      ]
    },
    {
      "name": "Vuln",
      "tools": [
        {
          "name": "nuclei",
          "desc": "Fast, customizable vulnerability scanner",
          "tags": ["vuln", "scanner"],
          "version": "3.0.0",
          "installed": true,
          "path": "/home/user/.nix-profile/bin/nuclei"
        }
      ]
    }
  ]
}
```

**响应字段（nix-build）：**

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `registry_mode` | string | 始终为 `"nix-build"` |
| `nix_installed` | boolean | Nix 包管理器是否已安装 |
| `categories` | array | 来自 flake.nix 的工具类别列表 |
| `categories[].name` | string | 类别名称（例如 "Subdomain"、"Vuln"） |
| `categories[].tools` | array | 此类别中的工具列表 |
| `categories[].tools[].name` | string | 二进制名称 |
| `categories[].tools[].desc` | string | 来自注册表的说明 |
| `categories[].tools[].tags` | []string | 来自注册表的标签 |
| `categories[].tools[].version` | string | 来自注册表的版本 |
| `categories[].tools[].repo_link` | string | 仓库 URL |
| `categories[].tools[].installed` | boolean | 二进制文件是否已安装 |
| `categories[].tools[].path` | string | 已安装二进制文件的完整路径 |

---

## 安装二进制文件或工作流

从注册表安装二进制文件，或从 git/zip URL 安装工作流。支持两种二进制安装模式。

**端点：** `POST /osm/api/registry-install`

**请求体：**

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `type` | string | **必填。** 值为 `"binary"` 或 `"workflow"` |
| `names` | []string | 要安装的二进制名称（用于 `type=binary`） |
| `install_all` | bool | 安装注册表中所有二进制文件（用于 `type=binary`） |
| `source` | string | Git URL、zip URL 或文件路径（用于 `type=workflow`） |
| `registry_url` | string | 自定义注册表 URL（可选，用于 `type=binary`） |
| `registry_mode` | string | `"direct-fetch"`（默认）或 `"nix-build"` |

---

### 安装二进制文件（直接获取模式）

直接从 GitHub 发布版或配置的 URL 下载二进制文件。

**安装特定二进制文件：**
```bash
curl -X POST http://localhost:8002/osm/api/registry-install \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "binary",
    "names": ["nuclei", "httpx", "ffuf"]
  }'
```

**安装注册表中所有二进制文件：**
```bash
curl -X POST http://localhost:8002/osm/api/registry-install \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "binary",
    "install_all": true
  }'
```

**响应：**
```json
{
  "message": "Binary installation completed",
  "registry_mode": "direct-fetch",
  "installed": ["nuclei", "httpx"],
  "installed_count": 2,
  "binaries_folder": "/home/user/osmedeus-base/binaries",
  "failed": [
    {"name": "ffuf", "error": "download failed"}
  ],
  "failed_count": 1
}
```

---

### 安装二进制文件（Nix 构建模式）

通过 Nix 包管理器使用 `nix profile add` 安装二进制文件。

**通过 Nix 安装特定二进制文件：**
```bash
curl -X POST http://localhost:8002/osm/api/registry-install \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "binary",
    "names": ["amass", "subfinder", "nuclei"],
    "registry_mode": "nix-build"
  }'
```

**安装所有 Nix 二进制文件：**
```bash
curl -X POST http://localhost:8002/osm/api/registry-install \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "binary",
    "install_all": true,
    "registry_mode": "nix-build"
  }'
```

**响应：**
```json
{
  "message": "Nix binary installation completed",
  "registry_mode": "nix-build",
  "installed": ["amass", "subfinder", "nuclei"],
  "installed_count": 3,
  "binaries_folder": "/home/user/osmedeus-base/binaries"
}
```

**错误（Nix 未安装）：**
```json
{
  "error": true,
  "message": "Nix is not installed. Install Nix first or use registry_mode=direct-fetch"
}
```

---

### 安装工作流

从 git 仓库或 zip 归档安装工作流。

**从 git URL 安装工作流：**
```bash
curl -X POST http://localhost:8002/osm/api/registry-install \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "workflow",
    "source": "https://github.com/osmedeus/osmedeus-workflow.git"
  }'
```

**从 zip URL 安装工作流：**
```bash
curl -X POST http://localhost:8002/osm/api/registry-install \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "workflow",
    "source": "https://example.com/custom-workflow.zip"
  }'
```

**响应：**
```json
{
  "message": "Workflow installed successfully",
  "source": "https://github.com/osmedeus/osmedeus-workflow.git",
  "workflow_folder": "/home/user/osmedeus-base/workflow"
}
```

# Extension Scanning — 插件扫描

插件扫描针对您的目标运行自定义 JavaScript 或 YAML 扩展模块。`extension` 阶段跳过所有内置的 Go 扫描器模块，只运行您的扩展，让您完全控制扫描逻辑。

## 快速入门

```bash
# 运行单个扩展脚本
vigolium run extension -t https://example.com --ext my-scanner.js

# 运行目录中的所有扩展
vigolium run extension -t https://example.com --ext-dir ./my-extensions/

# 与内置模块一起运行扩展（正常扫描 + 扩展）
vigolium scan -t https://example.com --ext my-scanner.js
```

## 插件阶段

当您使用 `--only extension`（或其别名 `ext`）时，Vigolium：

1. 跳过所有发现、爬虫、SPA 和导入阶段
2. 禁用所有内置的 Go 扫描器模块
3. 自动启用扩展
4. 在动态评估期间仅运行您的 JS/YAML 扩展模块

```bash
# 以下命令等效：
vigolium run extension -t https://example.com
vigolium scan -t https://example.com --only extension
vigolium scan -t https://example.com --only ext
```

## 加载扩展

### CLI 标志

| 标志 | 描述 |
|------|------|
| `--ext <path>` | 加载特定的扩展脚本（可重复） |
| `--ext-dir <dir>` | 覆盖扩展目录 |

两个标志都会自动启用扩展。它们可以与任何扫描命令组合使用。

```bash
# 加载多个扩展脚本
vigolium run extension -t https://example.com \
  --ext ./idor_detector.js \
  --ext ./header_leak.js

# 使用自定义扩展目录
vigolium run extension -t https://example.com --ext-dir ~/my-extensions/

# 混合使用：目录 + 额外脚本
vigolium scan -t https://example.com --ext-dir ~/extensions/ --ext ./extra-check.js
```

### 配置文件

在 `vigolium-configs.yaml` 中启用和配置扩展：

```yaml
dynamic-assessment:
  extensions:
    enabled: true
    extension_dir: "~/.vigolium/extensions/"   # 默认扩展目录
    custom_dir:                                 # 额外的脚本路径
      - ~/extra-extensions/custom-scanner.js
    variables:                                  # 脚本中可访问的自定义变量
      api_key: "your-key"
    allow_exec: false                           # 允许 shell 命令执行
    sandbox_dir: "~/.vigolium/sandbox/"         # exec 操作的沙箱目录
```

## 扩展类型

扩展接入扫描器流水线的四个点：

| 类型 | 运行时机 | 用途 |
|------|---------|------|
| `active` | 动态评估期间 | 发送载荷，检测漏洞 |
| `passive` | 分析捕获的流量 | 检查请求/响应，不产生新流量 |
| `pre_hook` | 每个请求发送前 | 修改请求，跳过资源，注入标头 |
| `post_hook` | 漏洞发现发出后 | 提升严重等级，丢弃误报 |

JavaScript 和 YAML 格式都支持所有四种类型。

## 管理扩展

```bash
# 列出已加载的扩展
vigolium extensions ls
vigolium ext ls

# 按类型过滤
vigolium ext ls --type active
vigolium ext ls --type passive

# 显示详细信息
vigolium ext ls -v

# 显示 API 文档
vigolium ext docs
vigolium ext docs --example    # 包含使用示例

# 安装预设扩展
vigolium ext preset            # 安装所有预设
vigolium ext preset idor       # 安装特定预设

# 评估临时 JavaScript
vigolium ext eval 'vigolium.log.info("hello")'
vigolium ext eval --ext-file script.js
echo 'vigolium.utils.md5("test")' | vigolium ext eval --stdin
```

## 预设扩展

Vigolium 附带了入门扩展预设。使用 `vigolium ext preset` 安装：

| 预设 | 类型 | 描述 |
|------|------|------|
| `reflected_param_scanner` | active | 检测响应中的反射参数 |
| `idor_detector` | active | 检测不安全的直接对象引用 |
| `ai_xss_scanner` | active | AI 增强的 XSS 扫描 |
| `sensitive_header_leak` | passive | 检测响应标头中的敏感信息 |
| `error_pattern_detector` | passive | 检测错误模式和堆栈跟踪 |
| `ai_false_positive_filter` | post_hook | AI 驱动的误报过滤 |
| `ai_response_analyzer` | passive (YAML) | AI 增强的响应分析 |
| `add_auth_header` | pre_hook | 注入授权标头 |
| `skip_static_assets` | pre_hook | 跳过静态资源 URL 的扫描 |
| `tag_critical_domains` | post_hook | 标记来自关键域的漏洞发现 |

预设安装到 `~/.vigolium/extensions/`。

## 扩展 vs 内置模块

| | `--only extension` | `--ext` 配合正常扫描 |
|-|-------------------|---------------------|
| 内置 Go 模块 | 禁用 | 启用 |
| 扩展模块 | 启用 | 启用 |
| 发现/爬虫 | 禁用 | 按策略 |
| 用例 | 隔离测试扩展 | 增强内置扫描 |

开发和测试扩展时使用 `--only extension`。在正常扫描上使用 `--ext` 以在内置模块之上添加扩展。

## 常见场景

```bash
# 开发和测试新扩展
vigolium run extension -t https://example.com --ext ./my-new-scanner.js

# 仅运行基于 YAML 的扩展
vigolium run extension -t https://example.com --ext-dir ./yaml-extensions/

# 扩展 + 完整平衡扫描
vigolium scan -t https://example.com --ext ./custom-check.js

# 扩展配合 OpenAPI 规范
vigolium scan -i api-spec.yaml -I openapi --only extension --ext ./api-fuzzer.js

# 扩展配合自定义变量
# （在 vigolium-configs.yaml 中设置变量，通过 vigolium.config.get("key") 访问）
vigolium run extension -t https://example.com --ext ./scanner-with-config.js

# 对单个 URL 运行扩展
vigolium scan-url https://example.com/api/users --ext ./idor_detector.js
```

## 延伸阅读

- [编写插件](../customization/writing-extensions.md) — 编写 JS 和 YAML 扩展的完整指南
- [扩展 API 参考](../api-references/extensions.md) — 完整的 `vigolium.*` API 接口
- `vigolium ext docs` — 内置 API 文档及示例

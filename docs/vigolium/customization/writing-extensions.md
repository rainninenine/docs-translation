# 编写插件

插件允许您向 Vigolium 添加自定义扫描逻辑，而无需修改核心扫描器。您可以使用 **JavaScript** 编写以获得完全的灵活性，使用 **YAML** 进行声明式模式匹配，或使用轻量级的 **快速检查** 和 **代码片段** 进行快速迭代。

---

## 目录

- [概述](#overview)
- [设置](#setup)
- [插件类型](#extension-types)
- [编写 JavaScript 插件](#writing-a-javascript-extension)
  - [主动模块](#active-module-js)
  - [被动模块](#passive-module-js)
  - [前置钩子](#pre-hook-js)
  - [后置钩子](#post-hook-js)
- [编写 YAML 插件](#writing-a-yaml-extension)
  - [主动模块](#active-module-yaml)
  - [被动模块](#passive-module-yaml)
  - [前置钩子](#pre-hook-yaml)
  - [后置钩子](#post-hook-yaml)
- [快速检查](#quick-checks)
- [代码片段](#snippets)
- [上下文对象参考](#context-objects-reference)
- [API 参考](#api-reference)
- [测试您的插件](#testing-your-extension)
- [配置参考](#configuration-reference)
- [提示与最佳实践](#tips-and-best-practices)

---

## 概述

插件在扫描管道的四个点接入：

| 类型 | 运行时机 | 用途 |
|---|---|---|
| `active` | 动态评估阶段 | 发送载荷，检测漏洞 |
| `passive` | 分析捕获的流量 | 检查请求/响应，不产生新流量 |
| `pre_hook` | 每个请求发送前 | 修改请求、跳过资源、注入头部 |
| `post_hook` | 漏洞发现发出后 | 升级严重性、丢弃误报 |

所有四种类型在 JS、YAML 以及（对于主动/被动）作为快速检查或代码片段中均受支持。YAML 适用于简单的模式匹配。JS 提供对 HTTP 请求、正则表达式、编码工具、数据库 API、OAST（带外）测试以及可选的 AI 增强分析的完全访问。快速检查和代码片段更轻量——非常适合由 Agent 生成或临时检查。

---

## 设置

### 1. 在配置中启用插件

在您的 `vigolium-configs.yaml` 中的 `dynamic-assessment` 下添加或取消注释 `extensions` 块：

```yaml
dynamic-assessment:
  extensions:
    enabled: true
    extension_dir: ~/.vigolium/extensions/   # 扫描此目录中的 .js 和 .vgm.yaml 文件
    custom_dir: []                           # 显式的额外路径
    variables:
      auth_token: "Bearer eyJ..."            # 可通过 vigolium.config.auth_token 访问
    limits:
      timeout: 30s
      max_memory_mb: 128
```

### 2. 放置您的插件文件

将任何 `.js` 或 `.vgm.yaml` 文件放入您的 `extension_dir`。Vigolium 会在下次扫描时自动发现它们。

### 3. 验证已加载

```bash
vigolium extensions ls
```

---

## 插件类型

### 模块导出契约（JS）

每个 JS 插件必须导出一个 `module.exports` 对象。必填字段：

| 字段 | 必填 | 描述 |
|---|---|---|
| `id` | 否（自动从文件名获取） | 唯一标识符 |
| `type` | **是** | `active`、`passive`、`pre_hook`、`post_hook` |
| `name` | 否 | 显示名称 |
| `description` | 否 | 插件功能描述 |
| `severity` | 对于主动/被动 | `critical`、`high`、`medium`、`low`、`info`、`suspect` |
| `confidence` | 否 | `tentative`、`firm`、`certain` |
| `scanTypes` | 对于主动/被动 | `["per_insertion_point"]`、`["per_request"]`、`["per_host"]` |
| `tags` | 否 | 用于 `--module-tag` 过滤的分类标签（例如 `["custom", "xss"]`） |

---

## 编写 JavaScript 插件

JS 插件在嵌入式 Sobek（ES5.1 兼容）VM 中运行。全局 `vigolium` 对象提供所有 API。

### 主动模块（JS）

主动模块发送修改后的请求以探测漏洞。在 `scanTypes` 中声明所需的扫描粒度：

- `per_insertion_point` — 每个参数（查询、正文、头部、Cookie）调用一次
- `per_request` — 每个请求/响应对调用一次
- `per_host` — 每个唯一主机名调用一次

**per_insertion_point 示例** — 检测反射输入：

```javascript
// 文件: ~/.vigolium/extensions/reflected_param_scanner.js
module.exports = {
  id: "reflected-param",
  name: "Reflected Parameter Scanner",
  type: "active",
  severity: "medium",
  confidence: "firm",
  tags: ["custom", "xss", "reflection"],
  scanTypes: ["per_insertion_point"],

  scanPerInsertionPoint: function(ctx, insertion) {
    // 生成唯一的金丝雀值
    var canary = "VGNM" + vigolium.utils.randomString(8);

    // 构建并发送注入金丝雀值的请求
    var req = insertion.buildRequest(canary);
    var resp = vigolium.http.send(req);

    if (!resp || !resp.body) return null;

    // 检查响应中是否出现金丝雀值
    if (resp.body.indexOf(canary) !== -1) {
      return [{
        matched: canary,
        url: ctx.request.url,
        name: "Reflected parameter: " + insertion.name,
        description: "Parameter '" + insertion.name + "' is reflected without encoding",
        severity: "medium"
      }];
    }
    return null;
  }
};
```

**per_request 示例** — 检测现有响应中的错误消息：

```javascript
// 文件: ~/.vigolium/extensions/error_pattern_detector.js
module.exports = {
  id: "error-pattern-detector",
  name: "Error Pattern Detector",
  type: "active",
  severity: "low",
  confidence: "firm",
  scanTypes: ["per_request"],

  scanPerRequest: function(ctx) {
    if (!ctx.response || !ctx.response.body) return null;

    var body = ctx.response.body;
    var patterns = [
      { regex: /Traceback \(most recent call last\)/i, name: "Python traceback" },
      { regex: /goroutine \d+ \[running\]/i,           name: "Go panic stack trace" },
      { regex: /SQLSTATE\[/i,                          name: "SQL error (SQLSTATE)" },
      { regex: /Fatal error:.*on line \d+/i,           name: "PHP fatal error" }
    ];

    var findings = [];
    for (var i = 0; i < patterns.length; i++) {
      if (patterns[i].regex.test(body)) {
        findings.push({
          matched: patterns[i].name,
          url: ctx.request.url,
          name: "Error pattern: " + patterns[i].name,
          description: "Response contains a " + patterns[i].name,
          severity: "low"
        });
      }
    }
    return findings.length > 0 ? findings : null;
  }
};
```

**主动/被动返回值：** 漏洞发现对象数组，如果未发现则返回 `null`。

每个漏洞发现对象：

```javascript
{
  matched: "...",        // 触发漏洞发现的内容（在输出中显示）
  url: "...",            // 完整 URL
  name: "...",           // 漏洞发现标题
  description: "...",    // 详细描述
  severity: "medium"     // 如果设置，则覆盖模块严重性
}
```

---

### 被动模块（JS）

被动模块分析现有的请求/响应对，而不发送新请求。添加 `scope` 字段以限制为 `"request"`、`"response"` 或 `"both"`（默认）。

```javascript
// 文件: ~/.vigolium/extensions/sensitive_header_leak.js
module.exports = {
  id: "sensitive-header-leak",
  name: "Sensitive Header Leak",
  type: "passive",
  severity: "info",
  confidence: "certain",
  scope: "response",
  scanTypes: ["per_request"],

  scanPerRequest: function(ctx) {
    if (!ctx.response || !ctx.response.headers) return null;

    var findings = [];
    var headers = ctx.response.headers;

    var poweredBy = headers["X-Powered-By"] || headers["x-powered-by"];
    if (poweredBy) {
      findings.push({
        matched: "X-Powered-By: " + poweredBy,
        url: ctx.request.url,
        name: "X-Powered-By header exposed",
        description: "Server technology revealed: " + poweredBy,
        severity: "info"
      });
    }

    return findings.length > 0 ? findings : null;
  }
};
```

---

### 前置钩子（JS）

前置钩子在每个请求发送到模块之前运行。返回修改后的请求、仅头部补丁，或返回 `null` 以完全跳过该请求。

```javascript
// 文件: ~/.vigolium/extensions/add_auth_header.js
module.exports = {
  id: "add-auth-header",
  name: "Auth Header Injector",
  type: "pre_hook",

  execute: function(request) {
    var token = vigolium.config.auth_token || "";
    if (token === "") {
      return request; // 原样传递
    }

    // 返回头部补丁——这些将被合并到现有请求中
    return {
      headers: {
        "Authorization": "Bearer " + token,
        "X-Correlation-ID": vigolium.utils.randomString(12)
      }
    };
  }
};
```

返回值选项：

| 返回值 | 效果 |
|---|---|
| `request`（未修改） | 原样传递 |
| `{ headers: {...} }` | 将这些头部合并到请求中 |
| `{ raw: "GET /..." }` | 替换整个原始请求 |
| `null` | 跳过此请求（模块不会看到它） |

**跳过静态资源示例：**

```javascript
// 文件: ~/.vigolium/extensions/skip_static_assets.js
module.exports = {
  id: "skip-static-assets",
  type: "pre_hook",

  execute: function(request) {
    var path = request.path || "";
    var skip = [".css", ".js", ".png", ".jpg", ".gif", ".svg", ".ico",
                ".woff", ".woff2", ".ttf", ".map"];
    for (var i = 0; i < skip.length; i++) {
      if (path.endsWith(skip[i])) return null;
    }
    return request;
  }
};
```

---

### 后置钩子（JS）

后置钩子接收每个发出的漏洞发现。返回（可能修改后的）结果，或返回 `null` 以抑制该漏洞发现。

```javascript
// 文件: ~/.vigolium/extensions/tag_critical_domains.js
module.exports = {
  id: "tag-critical-domains",
  name: "Critical Domain Tagger",
  type: "post_hook",

  execute: function(result) {
    if (!result || !result.url) return result;

    var url = result.url.toLowerCase();
    var critical = ["payment", "admin", "auth", "checkout", "billing"];

    for (var i = 0; i < critical.length; i++) {
      if (url.indexOf(critical[i]) !== -1) {
        var sev = result.info ? result.info.severity : "info";
        var escalated = { info: "low", low: "medium", medium: "high", high: "critical" }[sev] || sev;

        return {
          url: result.url,
          matched: result.matched,
          info: {
            name: result.info.name + " [CRITICAL: " + critical[i] + "]",
            description: result.info.description,
            severity: escalated
          }
        };
      }
    }
    return result;
  }
};
```

---

## 编写 YAML 插件

YAML 插件（`.vgm.yaml`）是常见模式的声明式替代方案。它们不需要编程知识，并编译为与 JS 插件相同的内部模块接口。

### 主动模块（YAML）

使用 `rules` 定义匹配-然后-发出的对。每个规则指定一个匹配条件和匹配时发出的漏洞发现。

```yaml
# 文件: ~/.vigolium/extensions/error_patterns.vgm.yaml
id: error-pattern-detector-yaml
name: Error Pattern Detector (YAML)
description: Detects stack traces and error messages in responses
type: active
severity: low
confidence: firm
tags: [custom, error-detection]
scan_types:
  - per_request

rules:
  - match:
      body_regex: "(?i)Traceback \\(most recent call last\\)"
    finding:
      name: "Error pattern: Python traceback"
      description: "Response body contains a Python traceback"
      severity: low

  - match:
      body_regex: "(?i)goroutine \\d+ \\[running\\]"
    finding:
      name: "Error pattern: Go panic stack trace"
      description: "Response body contains a Go panic stack trace"
      severity: low

  - match:
      body_regex: "(?i)SQLSTATE\\["
    finding:
      name: "Error pattern: SQL error"
      description: "Response body contains a SQL SQLSTATE error"
      severity: low
```

**顶层主动字段：**

| 字段 | 描述 |
|---|---|
| `tags` | 用于 `--module-tag` 过滤的分类标签 |
| `scan_types` | `per_insertion_point`、`per_request`、`per_host` |
| `payloads` | 要注入的字符串列表（与 `per_insertion_point` 一起使用） |
| `matchers` | `MatcherDef` 列表——全部应用于同一个漏洞发现 |
| `matchers_condition` | `or`（默认）或 `and`——匹配器如何组合 |
| `finding` | 匹配器通过时发出的单个漏洞发现 |
| `rules` | `{match, finding}` 对列表——独立评估 |

当不同模式应发出不同漏洞发现时使用 `rules`。当所有条件必须同时为真时使用 `matchers` + `finding`。

**匹配器类型：**

```yaml
matchers:
  # 检查正文是否包含字符串
  - contains: "password"

  # 使用正则表达式检查正文
  - regex: "(?i)error|exception"

  # 检查响应头部是否存在
  - type: header
    name: X-Powered-By

  # 检查 HTTP 状态码
  - type: status
    codes: [500, 502, 503]

  # 否定条件
  - contains: "success"
    negate: true
```

---

### 被动模块（YAML）

被动 YAML 模块使用相同的 `rules` 结构，但不发送新请求：

```yaml
# 文件: ~/.vigolium/extensions/sensitive_headers.vgm.yaml
id: sensitive-header-leak-yaml
name: Sensitive Header Leak (YAML)
type: passive
severity: info
confidence: certain
scope: response          # request | response | both
scan_types:
  - per_request

rules:
  - match:
      response_header: X-Powered-By
    finding:
      name: X-Powered-By header exposed
      description: "Server technology revealed via X-Powered-By header"
      matched: "{{matched}}"  # 插值匹配到的头部值
      severity: info

  - match:
      response_header: Server
      regex: "[0-9]+\\.[0-9]+"   # 仅当值包含版本号时匹配
    finding:
      name: Server version disclosed
      description: "Server header exposes version information"
      matched: "{{matched}}"
      severity: low
```

**`rules[].match` 的规则匹配字段：**

| 字段 | 描述 |
|---|---|
| `body_contains` | 响应正文包含字符串 |
| `body_regex` | 响应正文匹配正则表达式 |
| `response_header` | 响应头部名称存在 |
| `regex` | 应用于头部值的附加正则表达式 |
| `contains` | 头部值必须包含的字符串 |
| `status` | HTTP 状态码列表 |

---

### 前置钩子（YAML）

YAML 中的前置钩子支持头部注入、扩展名跳过和条件跳过。

**注入头部：**

```yaml
# 文件: ~/.vigolium/extensions/add_auth.vgm.yaml
id: add-auth-header-yaml
name: Auth Header Injector (YAML)
type: pre_hook

# 如果配置变量未设置，则跳过此钩子
skip_when:
  config_empty: auth_token

add_headers:
  Authorization: "Bearer {{config.auth_token}}"
  X-Correlation-ID: "{{rand(12)}}"
```

**跳过静态文件：**

```yaml
# 文件: ~/.vigolium/extensions/skip_static.vgm.yaml
id: skip-static-assets-yaml
name: Static Asset Skipper (YAML)
type: pre_hook

skip_extensions:
  - .css
  - .js
  - .png
  - .jpg
  - .gif
  - .svg
  - .ico
  - .woff
  - .woff2
  - .ttf
  - .map
```

**前置钩子 YAML 字段：**

| 字段 | 描述 |
|---|---|
| `add_headers` | 头部名称 → 要注入的值的映射 |
| `skip_extensions` | 导致请求被跳过的 URL 路径后缀 |
| `skip_when.config_empty` | 如果此配置变量为空则跳过 |
| `skip_when.url_contains` | 如果 URL 包含这些字符串中的任何一个则跳过 |

**头部值中可用的模板函数：**

| 模板 | 展开为 |
|---|---|
| `{{config.VAR_NAME}}` | 用户定义的配置变量 |
| `{{rand(N)}}` | 长度为 N 的随机字母数字字符串 |

---

### 后置钩子（YAML）

YAML 中的后置钩子可以根据 URL 模式升级严重性或丢弃漏洞发现。

**为关键路径升级严重性：**

```yaml
# 文件: ~/.vigolium/extensions/critical_tagger.vgm.yaml
id: tag-critical-domains-yaml
name: Critical Domain Tagger (YAML)
type: post_hook

escalate:
  when_url_contains:
    - payment
    - admin
    - auth
    - checkout
    - billing
  tag: "CRITICAL"
  bump_severity: true      # info→low, low→medium, medium→high, high→critical
```

**丢弃某些路径上的低严重性漏洞发现：**

```yaml
id: drop-noisy-findings
type: post_hook

drop_when:
  severity:
    - info
  url_contains:
    - /static/
    - /assets/
```

---

## 快速检查

快速检查是最轻量的插件格式——声明式 JSON 对象，定义“发送载荷，检查响应”模式，无需 JavaScript。它们非常适合由 Agent 生成的检查和快速迭代。

### 每个插入点

将载荷注入每个参数并检查响应：

```json
{
  "id": "ssti-jinja2",
  "severity": "high",
  "scan": "per_insertion_point",
  "payloads": ["{{7*7}}", "${7*7}", "<%=7*7%>"],
  "match": {"body_contains": "49"}
}
```

### 每个请求 / 每个主机

发送特定请求并检查响应：

```json
{
  "id": "debug-endpoint",
  "severity": "medium",
  "scan": "per_host",
  "requests": [
    {"method": "GET", "path": "/.env"},
    {"method": "GET", "path": "/debug/vars"}
  ],
  "match": {"status": 200, "body_regex": "(DB_PASSWORD|SECRET_KEY)"}
}
```

### 匹配字段

匹配条件使用 OR 逻辑：

| 字段 | 描述 |
|---|---|
| `body_contains` | 响应正文包含字符串 |
| `body_regex` | 响应正文匹配正则表达式 |
| `status` | HTTP 状态码等于值 |
| `header_contains` | 响应头部包含字符串 |

### 规则

- `id` 必须为小写字母，使用连字符（例如 `"ssti-jinja2"`）
- `scan` 为以下之一：`per_insertion_point`、`per_request`、`per_host`
- `severity` 为以下之一：`critical`、`high`、`medium`、`low`、`info`
- 快速检查在运行时自动包装为完整的扩展模块

---

## 代码片段

代码片段是快速检查和完整扩展之间的中间地带——您只需编写**函数体**（无需样板代码），它会自动包装在模块框架中。当您需要自定义逻辑或 `vigolium.*` API 访问时，使用代码片段。

### 格式

```json
{
  "id": "idor-check",
  "severity": "high",
  "scan": "per_request",
  "body": "var related = vigolium.db.records.getRelated(ctx.record.uuid);\nvar cmp = vigolium.db.compareResponses(related);\nif (!cmp.all_similar) {\n  return [{url: ctx.request.url, matched: 'Response variance', name: 'Potential IDOR'}];\n}\nreturn null;"
}
```

### 可用变量

在代码片段体内，您可以访问：

| 变量 | 描述 |
|---|---|
| `ctx` | 请求/响应上下文（`ctx.request`、`ctx.response`、`ctx.record`） |
| `insertion` | 插入点对象（仅适用于 `per_insertion_point` 扫描类型） |
| `vigolium.http` | HTTP 请求、会话、批量、重放、序列、身份验证测试 |
| `vigolium.db` | 用于记录和漏洞发现的数据库查询 |
| `vigolium.utils` | 编码、哈希、差异、JWT、CSS 选择器等 |
| `vigolium.parse` | URL、请求、响应、HTML 解析 |
| `vigolium.scan` | 模块列表、范围、漏洞发现创建 |
| `vigolium.source` | 源代码访问和搜索 |
| `vigolium.agent` | AI 增强分析 |
| `vigolium.oast` | 带外测试 |
| `vigolium.ingest` | 数据导入 |
| `vigolium.payloads()` | 内置载荷词表 |

### 规则

- `id` 必须为小写字母，使用连字符
- `scan` 为以下之一：`per_insertion_point`、`per_request`、`per_host`
- `body` 包含函数体作为字符串（换行符转义为 `\n`）
- 返回值遵循与完整扩展相同的约定：漏洞发现数组或 `null`

---

## 上下文对象参考

### `ctx` — 传递给所有主动/被动模块函数

```javascript
ctx.request.url        // "https://example.com/api/users?id=1"
ctx.request.method     // "GET"
ctx.request.path       // "/api/users"
ctx.request.hostname   // "example.com"
ctx.request.headers    // { "Content-Type": "application/json", ... }
ctx.request.raw        // 完整的原始 HTTP 请求字符串

ctx.response.status    // 200
ctx.response.body      // 响应正文作为字符串
ctx.response.headers   // { "X-Powered-By": "PHP/8.1", ... }
ctx.response.raw       // 完整的原始 HTTP 响应字符串
ctx.response.title     // HTML 页面标题（如果适用）
```

### `insertion` — `scanPerInsertionPoint` 的第二个参数

```javascript
insertion.name              // "id"（参数名称）
insertion.baseValue         // "1"（原始值）
insertion.type              // "url_param" | "body_param" | "header" | "cookie"
insertion.buildRequest(val) // 返回注入 val 后的原始请求字符串
```

### `ctx.record` — 当前 HTTP 记录上下文

```javascript
ctx.record.uuid              // 当前记录的数据库 UUID
ctx.record.annotate(patch)   // 更新 risk_score/remarks
ctx.record.addRiskScore(delta) // 增加风险分数（可为负数，最小值为 0）
ctx.record.addRemarks(remarks) // 追加备注（去重）
```

---

## API 参考

### vigolium.log

| 函数 | 描述 |
|---|---|
| `info(msg)` | 记录信息消息 |
| `warn(msg)` | 记录警告消息 |
| `error(msg)` | 记录错误消息 |
| `debug(msg)` | 记录调试消息 |

### vigolium.utils

**编码：**

| 函数 | 描述 |
|---|---|
| `base64Encode(s)` / `base64Decode(s)` | Base64 编码/解码 |
| `urlEncode(s)` / `urlDecode(s)` | URL 编码/解码 |
| `htmlEncode(s)` / `htmlDecode(s)` | HTML 实体编码/解码 |

**哈希：**

| 函数 | 描述 |
|---|---|
| `sha1(s)` | SHA-1 哈希 |
| `sha256(s)` | SHA-256 哈希 |
| `md5(s)` | MD5 哈希 |

**随机：**

| 函数 | 描述 |
|---|---|
| `randomString(len)` | 随机字母数字字符串 |

**正则表达式：**

| 函数 | 描述 |
|---|---|
| `regexMatch(str, pattern)` | 测试字符串是否匹配正则表达式 |
| `regexExtract(str, pattern)` | 从字符串中提取匹配项 |

**文件 I/O：**

| 函数 | 描述 |
|---|---|
| `readFile(path)` | 读取文件内容 |
| `readLines(path)` | 将文件读取为行数组 |
| `writeFile(path, data)` | 将数据写入文件 |
| `mkdir(path)` | 创建目录 |
| `glob(pattern)` | 查找匹配模式的文件 |

**URL 工具：**

| 函数 | 描述 |
|---|---|
| `parse_url(url, format)` | 使用格式解析 URL |
| `pathToTemplate(path)` | 将动态段替换为 `*` |
| `hasDynamicSegment(path)` | 检查动态段（ID、UUID） |

**参数工具：**

| 函数 | 描述 |
|---|---|
| `toSet(csv)` | 将 CSV 字符串转换为 `{key: true}` 映射 |
| `extractParamNames(str)` | 从查询/正文中提取去重的参数名称 |

**差异和相似度：**

| 函数 | 描述 |
|---|---|
| `diff(a, b)` | 逐行比较 → `{added, removed, similarity}` |
| `similarity(a, b)` | 基于词令牌的 Jaccard 相似度（0.0–1.0） |
| `diffResponses(a, b)` | 结构化 HTTP 响应比较 → `{status_match, body_similarity, header_diff, body_diff, length_diff, likely_same_content}` |

**HTML：**

| 函数 | 描述 |
|---|---|
| `cssSelect(html, selector)` | CSS 选择器查询 → `[{text, attrs, html}]` |

**令牌提取：**

| 函数 | 描述 |
|---|---|
| `extractToken(response, rules)` | 使用可配置规则（json/header/cookie/regex）从 HTTP 响应中提取令牌 |

**JWT：**

| 函数 | 描述 |
|---|---|
| `jwtDecode(token)` | 解码 JWT 而不验证 → `{header, payload, signature}` |
| `jwtEncode(payload, opts?)` | 伪造 JWT（HS256/HS384/HS512/none） |
| `jwtExpired(token)` | 检查 JWT 是否过期 |

**Multipart：**

| 函数 | 描述 |
|---|---|
| `multipart(fields)` | 构建 multipart/form-data 正文 → `{body, contentType}` |

**异常检测：**

| 函数 | 描述 |
|---|---|
| `detectAnomaly(responses)` | 通过差异对响应评分 → `[{index, score}]` |

**其他：**

| 函数 | 描述 |
|---|---|
| `sleep(ms)` | 休眠毫秒数 |
| `exec(cmd)` | 执行 shell 命令（需要 `allow_exec`）→ `{stdout, stderr, exitCode}` |
| `getEnv(name)` / `setEnv(name, value)` | 环境变量 |
| `jsonExtract(json, path)` | 按路径从 JSON 中提取值 |

### vigolium.parse

| 函数 | 描述 |
|---|---|
| `url(str)` | 解析 URL → `{scheme, host, hostname, port, path, query, fragment, params, segments, template}` |
| `request(raw)` | 解析原始 HTTP 请求 → `{method, path, query, version, headers, body, host, params, cookies}` |
| `response(raw)` | 解析原始 HTTP 响应 → `{status, statusText, version, headers, body, cookies, contentType}` |
| `headers(str)` | 解析头部块 → `{name: value}` |
| `cookies(str)` | 解析 Cookie 头部 → `{name: value}` |
| `query(str)` | 解析查询字符串 → `{name: value}` |
| `json(str)` | 解析 JSON 字符串 → 原生值 |
| `form(body)` | 解析 URL 编码表单 → `{name: value}` |
| `html(str)` | 解析 HTML → `{forms, links, scripts, meta}` |

### vigolium.http

**基本请求：**

| 函数 | 描述 |
|---|---|
| `get(url, opts?)` | HTTP GET |
| `post(url, body, opts?)` | HTTP POST |
| `request(opts)` | 完全控制（method、url、headers、body） |
| `send(rawRequest)` | 发送原始 HTTP 请求字符串 |
| `buildRequest(rawRequest, overrides)` | 克隆并修改原始请求（method、path、headers、body、query） |

**会话：**

| 函数 | 描述 |
|---|---|
| `session(opts?)` | 创建持久会话，共享 cookie jar 和默认头部 |
| `login(opts)` | 发送凭据，提取身份验证令牌，返回会话 |
# 扫描范围 — 模块如何分发

每个扫描器模块声明一个**扫描范围**，告知执行器*何时*以及*多久*调用一次该模块。扫描范围决定了模块操作的粒度：按参数、按请求或按主机。

```go
type ScanScope uint8

const (
    ScanScopeInsertionPoint ScanScope = 1 << iota  // 按参数
    ScanScopeRequest                                // 按请求
    ScanScopeHost                                   // 按主机
)
```

范围是一个位掩码。一个模块可以声明多个范围（例如 `ScanScopeRequest | ScanScopeInsertionPoint`）。

---

## 概览图

```
                        Incoming HttpRequestResponse
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │         Executor               │
                    │  (scope filtering + dispatch)  │
                    └───────┬───────┬───────┬────────┘
                            │       │       │
              ┌─────────────┘       │       └──────────────┐
              ▼                     ▼                      ▼
   ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────┐
   │  ScanScopeHost   │   │ ScanScopeRequest  │   │ScanScopeInsertion│
   │                  │   │                   │   │     Point        │
   │  每个唯一主机     │   │  每个唯一请求     │   │  请求中每个参数   │
   │  运行一次         │   │  运行一次         │   │  运行一次         │
   │                  │   │                   │   │                  │
   │  例如 example.com│   │  例如 GET /api?id=1│   │  例如 ?a=1&b=2&c=3│
   │  即使有500个请求  │   │  只运行1次        │   │  运行3次          │
   │  也只运行1次      │   │                   │   │                  │
   └─────────────────┘   └──────────────────┘   └──────────────────┘
          │                       │                       │
          ▼                       ▼                       ▼
   CORS 配置错误          403 绕过            SQLi 在参数 "a" 上
   默认凭据               Host 头注入         SQLi 在参数 "b" 上
   敏感文件               JWT 操纵            SQLi 在参数 "c" 上
   GraphQL 内省          方法篡改             SSTI 在参数 "a" 上
                           缓存投毒           SSTI 在参数 "b" 上
                                                 ...
```

---

## 三种范围

### ScanScopeInsertionPoint

**对请求中的每个参数（插入点）调用一次。**

执行器解析原始 HTTP 请求，提取每个可注入的位置，并逐个传递给模块。模块接收一个 `InsertionPoint`，其中包含 `BuildRequest(payload)` 方法，用于在该精确位置注入其载荷。

#### 逐步工作原理

给定以下请求：

```http
POST /api/search?lang=en HTTP/1.1
Host: example.com
Cookie: session=abc123
Content-Type: application/x-www-form-urlencoded

query=test&page=1
```

**步骤 1 — 执行器调用 `CreateAllInsertionPoints()` 并找到 5 个插入点：**

```
┌─────┬──────────────┬────────────┬──────────┐
│  #  │  Name        │  Type      │  Value   │
├─────┼──────────────┼────────────┼──────────┤
│  1  │  lang        │  URL_PARAM │  en      │
│  2  │  session     │  COOKIE    │  abc123  │
│  3  │  query       │  BODY_PARAM│  test    │
│  4  │  page        │  BODY_PARAM│  1       │
│  5  │  Host        │  HEADER    │  exampl… │
└─────┴──────────────┴────────────┴──────────┘
```

**步骤 2 — 对于每个插入点，执行器并行运行所有兼容的模块：**

```
插入点 #1: lang=en (URL_PARAM)
  ├── sqli_error_based  →  lang=' OR 1=1--     → 检查响应中是否有 SQL 错误
  ├── ssti_detection    →  lang={{7*7}}         → 检查响应中是否包含 "49"
  ├── lfi_generic       →  lang=../../etc/passwd → 检查响应中是否包含 "root:"
  ├── crlf_injection    →  lang=%0d%0aX:injected → 检查响应头
  └── ... (所有接受 URL_PARAM 的 PER_INSERTION_POINT 模块)

插入点 #2: session=abc123 (COOKIE)
  ├── sqli_error_based  →  session=' OR 1=1--  → 检查响应
  ├── ssti_detection    →  session={{7*7}}      → 检查响应
  └── ... (仅接受 COOKIE 类型的模块)

插入点 #3: query=test (BODY_PARAM)
  ├── sqli_error_based  →  query=' OR 1=1--    → 检查响应
  ├── ssti_detection    →  query={{7*7}}        → 检查响应
  ├── ssrf_detection    →  query=http://burp.co → 检查 OOB 回调
  └── ...

... 以此类推，每个插入点
```

**步骤 3 — 每个模块使用 `BuildRequest(payload)` 构造修改后的请求。** 只有目标参数发生变化；其他所有内容保持不变：

```
原始:  POST /api/search?lang=en HTTP/1.1  ...  query=test&page=1
注入:  POST /api/search?lang=en HTTP/1.1  ...  query=' OR 1=1--&page=1
                                                          ^^^^^^^^^^^^
                                                    仅此部分改变
```

**哪些算作插入点：**

| 类型 | 示例 | 测试内容 |
|------|------|----------|
| URL 参数 | `?id=123` | 值 `123` |
| 请求体参数 | `username=admin` | 值 `admin` |
| JSON 值 | `{"user":"admin"}` | 值 `"admin"` |
| Cookie | `session=abc` | 值 `abc` |
| HTTP 头 | `Host: example.com` | 值 `example.com` |
| XML 元素 | `<id>5</id>` | 值 `5` |
| XML 属性 | `<tag attr="val">` | 值 `val` |
| 多部分字段 | `Content-Disposition: name="file"` | 字段值 |
| URL 路径文件夹 | `/api/users/123` | 段 `123` |
| URL 路径文件名 | `/api/report.pdf` | 文件名 `report.pdf` |
| 参数名 (URL) | `?id=123` | 键 `id` 本身 |
| 参数名 (请求体) | `username=admin` | 键 `username` 本身 |
| 整个请求体 | 完整 POST 体 | 整个请求体作为一个整体 |

当 `includeNested=true`（默认）时，执行器还会发现**嵌套结构**——例如，一个 URL 参数的值是 Base64 编码的 JSON，将为该 JSON 中的每个键生成额外的插入点。

每个模块还声明它接受哪些 `InsertionPointType`（通过 `AllowedInsertionPointTypes()`），因此 SQLi 模块可能只测试 URL 参数、请求体参数和 JSON 值，而跳过 Cookie 和头。

**典型发现的漏洞：**
- SQL 注入（基于错误、盲注）
- 跨站脚本（XSS）
- 服务器端模板注入（SSTI）
- 命令注入
- 路径遍历 / LFI
- SSRF
- CRLF 注入
- NoSQL 注入
- XML/SAML 注入
- 不安全反序列化

**使用此范围的活跃模块（18 个）：**
`sqli_error_based`, `ssti_detection`, `reflected_ssti`, `csti_detection`, `ssrf_detection`, `lfi_generic`, `lfi_path_traversal`, `crlf_injection`, `nosqli_error_based`, `nosqli_operator_injection`, `xml_saml_security`, `insecure_deserialization`, `backslash_transformation`, `suspect_transform`, `smart_behavior_detection`, `input_behavior_probe`, `race_interference`, `oast_probe`（混合）

---

### ScanScopeRequest

**对每个唯一的请求/响应对调用一次。**

模块接收整个 `HttpRequestResponse`，并自行决定修改什么。它不会被赋予特定参数——它对请求结构拥有完全控制权。

此范围用于以下漏洞：
- 不映射到单个参数（例如，更改 HTTP 方法、添加新头）
- 需要跨参数上下文（例如，比较多个参数之间的时序）
- 测试请求级属性（例如，JWT 令牌、CSRF 令牌、缓存行为）
- 内部管理自己的参数迭代以实现专门逻辑

#### 逐步工作原理

给定一个返回 403 的请求：

```http
GET /admin/dashboard HTTP/1.1
Host: example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

**`forbidden_bypass` 模块接收整个请求并自行尝试多种攻击向量：**

```
原始请求 → 403 Forbidden

尝试 1: 路径技巧
  GET /./admin/dashboard       → 403  (未绕过)
  GET /admin/dashboard/./      → 403  (未绕过)
  GET /admin/dashboard..;/     → 200  ← 绕过成功！

尝试 2: 头注入（如果路径技巧失败）
  GET /anything
  X-Original-URL: /admin/dashboard  → 检查状态码

尝试 3: 方法篡改（如果头注入失败）
  PUT /admin/dashboard         → 检查状态码
  PATCH /admin/dashboard       → 检查状态码
  DELETE /admin/dashboard      → 检查状态码

尝试 4: 方法覆盖头
  POST /admin/dashboard
  X-HTTP-Method-Override: GET  → 检查状态码
```

**`host_header_injection` 模块测试头反射：**

```
原始:
  Host: example.com                     → 响应体

测试 1:
  Host: evil.attacker.com               → "evil.attacker.com" 是否出现在响应中？

测试 2:
  X-Forwarded-Host: evil.attacker.com   → 是否在 Location 头中反射？

测试 3:
  Forwarded: host=evil.attacker.com     → 是否在任何地方反射？
```

**`jwt_vulnerability` 模块操纵 JWT 令牌：**

```
原始令牌:    eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiam9obiJ9.signature
                   │                     │                      │
                   header                payload                signature

测试 1 — 算法混淆：
  更改 alg: HS256 → none，发送不带签名的令牌 → 是否仍被接受？

测试 2 — 弱密钥：
  尝试使用常见密钥签名（"secret", "password", ""）→ 是否被接受？
```

注意，这些攻击都不针对单个参数——它们修改请求结构、头、方法或令牌。这就是为什么它们使用 `ScanScopeRequest` 而不是 `ScanScopeInsertionPoint`。

**典型发现的漏洞：**
- 403/401 绕过（路径技巧、方法篡改、头注入）
- Host 头注入
- 开放重定向
- JWT 漏洞（算法混淆、弱密钥）
- CSRF 验证绕过
- Web 缓存投毒
- 原型污染
- XXE（全请求体注入）
- HTTP 方法覆盖
- Swagger/API 文档暴露
- 文件上传漏洞
- JSONP 回调注入
- Nginx 路径转义

**使用此范围的活跃模块（20 个）：**
`forbidden_bypass`, `host_header_injection`, `open_redirect`, `jwt_vulnerability`, `csrf_verify`, `web_cache_poisoning`, `prototype_pollution`, `client_prototype_pollution`, `xxe_generic`, `xss_light_scanner`（3 个子模块）, `sqli_boolean_blind`, `code_exec`, `file_upload_scan`, `spring_actuator_misconfig`, `nginx_path_escape`, `path_normalization`, `jsonp_callback`, `oast_probe`（混合）

**使用此范围的被动模块（19 个）：**
`info_disclosure_detect`, `secret_detect`, `cookie_security_detect`, `cors_headers_detect`, `content_type_mismatch`, `dom_xss_detect`, `csrf_detect`, `mixed_content_detect`, `auth_headers_detect`, `jwt_weak_secret`, `oauth_facebook_detect`, `openredirect_params`, `sensitive_url_params`, `sourcemap_detect`, `sql_syntax_detect`, `serialized_object_detect`, `crypto_weakness_detect`, `anomaly_ranking`, `idor_params_detect`

---

### ScanScopeHost

**对每个唯一主机调用一次。**

执行器按主机名去重——如果 500 个请求到达 `example.com`，模块只运行一次。此范围适用于昂贵的单次检查，适用于整个主机，而不是单个页面或参数。

#### 逐步工作原理

给定 500 个对 `example.com` 的不同请求：

```
请求 #1:   GET /api/users HTTP/1.1        Host: example.com
请求 #2:   POST /api/login HTTP/1.1       Host: example.com
请求 #3:   GET /products?id=42 HTTP/1.1   Host: example.com
...
请求 #500: GET /about HTTP/1.1            Host: example.com
```

**执行器仅在第一次看到该主机的请求时运行每个 `ScanScopeHost` 模块。其余 499 个请求被跳过：**

```
请求 #1 到达 example.com（首次看到）
  ├── sensitive_file_discovery
  │     GET /.env               → 404
  │     GET /.git/config        → 200 ← 发现！
  │     GET /robots.txt         → 200
  │     GET /.DS_Store          → 404
  │     GET /wp-config.php      → 404
  │
  ├── cors_misconfiguration
  │     GET / with Origin: https://evil.com
  │     → 检查 Access-Control-Allow-Origin 头
  │     → 反射了 "evil.com"？← CORS 配置错误！
  │
  ├── default_credentials
  │     POST /login with admin:admin      → 401
  │     POST /login with admin:password   → 401
  │     POST /login with root:root        → 200 ← 默认凭据！
  │
  ├── graphql_scan
  │     POST /graphql with introspection query
  │     → 返回了模式？← 内省已启用！
  │
  └── http_request_smuggling
        CL.TE 和 TE.CL 不同步探测

请求 #2-500 到达 example.com
  └── 主机已见过 → 所有 ScanScopeHost 模块被跳过
```

**典型发现的漏洞：**
- CORS 配置错误
- 默认凭据
- GraphQL 内省已启用
- HTTP 请求走私
- 敏感文件发现（`.env`, `.git/config`, `robots.txt`）
- 缺少安全头

**使用此范围的活跃模块（5 个）：**
`cors_misconfiguration`, `default_credentials`, `graphql_scan`, `http_request_smuggling`, `sensitive_file_discovery`

**使用此范围的被动模块（1 个）：**
`security_headers_missing`

---

## 混合范围

一个模块可以声明多个范围。目前，**`oast_probe`** 是唯一这样做的模块：

```go
ScanScopeRequest | ScanScopeInsertionPoint
```

这意味着它以两种模式运行以获得最大覆盖：

```
请求: POST /api/webhook?url=https://example.com HTTP/1.1
         Host: target.com
         X-Callback: https://app.internal

─── 作为 ScanScopeInsertionPoint ───

  插入点 #1: url=https://example.com (URL_PARAM)
    → url=https://<oast-callback-id>.oast.vigolium.io
    → 等待 DNS/HTTP 回调...

  插入点 #2: X-Callback (HEADER)
    → X-Callback: https://<oast-callback-id>.oast.vigolium.io
    → 等待 DNS/HTTP 回调...

─── 作为 ScanScopeRequest ───

  替换 Host 头:
    → Host: <oast-callback-id>.oast.vigolium.io
    → 等待 DNS/HTTP 回调...

  添加 Referer 头:
    → Referer: https://<oast-callback-id>.oast.vigolium.io
    → 等待 DNS/HTTP 回调...
```

OAST 回调可能来自参数级向量（通过 URL 参数的 SSRF）或请求级向量（通过 Host 头的盲 SSRF），因此它需要两种范围。

---

## 不同输入下的行为

### 带参数的完整请求

```http
POST /api/login?ref=home HTTP/1.1
Host: example.com
Cookie: lang=en
Content-Type: application/json

{"username":"admin","password":"secret"}
```

```
┌──────────────────┬──────────────────────────────────────────────────────────┐
│ 范围             │ 行为                                                    │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ InsertionPoint   │ 找到 5 个插入点：                                      │
│                  │   1. ref=home           (URL_PARAM)                     │
│                  │   2. lang=en            (COOKIE)                        │
│                  │   3. username=admin     (JSON_PARAM)                    │
│                  │   4. password=secret    (JSON_PARAM)                    │
│                  │   5. Host=example.com   (HEADER)                        │
│                  │ 每个由所有兼容模块测试（SQLi, SSTI, …）                │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Request          │ 完整请求传递给每个模块。测试：                          │
│                  │   - 方法篡改（POST → PUT, DELETE）                     │
│                  │   - 头中的 JWT？算法混淆                               │
│                  │   - CSRF 令牌存在？尝试移除它                          │
│                  │   - Host 头反射                                        │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Host             │ 首次看到 example.com？                                 │
│                  │   是 → 敏感文件探测、CORS 检查等                       │
│                  │   否 → 完全跳过                                        │
└──────────────────┴──────────────────────────────────────────────────────────┘
```

### 无参数的简单 URL

```http
GET /admin HTTP/1.1
Host: example.com
```

```
┌──────────────────┬──────────────────────────────────────────────────────────┐
│ 范围             │ 行为                                                    │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ InsertionPoint   │ CreateAllInsertionPoints 返回空列表。                   │
│                  │ 无查询参数、无请求体、无 Cookie。                       │
│                  │ *** 所有插入点模块被跳过 ***                            │
│                  │ 不会进行 SQLi、XSS、SSTI 或注入测试。                  │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Request          │ 模块仍然运行：                                          │
│                  │   - forbidden_bypass: 响应是 403？                      │
│                  │     尝试 /./admin, /admin..;/, PUT /admin               │
│                  │   - host_header_injection: 交换 Host 头                 │
│                  │   - path_normalization: /admin vs /Admin vs /ADMIN      │
│                  │   - 敏感文件、swagger 发现等                            │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Host             │ 同上 — 如果主机是新的，运行一次。                      │
└──────────────────┴──────────────────────────────────────────────────────────┘
```

### 静态文件 URL

```http
GET /assets/style.css HTTP/1.1
Host: example.com
```

```
┌──────────────────┬──────────────────────────────────────────────────────────┐
│ 范围             │ 行为                                                    │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ InsertionPoint   │ 空插入点列表。跳过。                                    │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Request          │ 活跃模块：大多数通过 CanProcess() 跳过 — 默认过滤      │
│                  │   掉 .css/.js/.png/.jpg 扩展名。                        │
│                  │ 被动模块：仍然运行 — 例如 secret_detect 检查泄露的     │
│                  │   API 密钥，sourcemap_detect 在 JS 包中查找 .map 引用。 │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Host             │ 如果主机是新的，运行一次。                              │
└──────────────────┴──────────────────────────────────────────────────────────┘
```

### 仅包含路径参数的 URL（REST 风格）

```http
GET /api/users/42/profile HTTP/1.1
Host: example.com
```

```
┌──────────────────┬──────────────────────────────────────────────────────────┐
│ 范围             │ 行为                                                    │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ InsertionPoint   │ 路径段被提取为插入点：                                  │
│                  │   1. "42"      (PATH_FOLDER)                           │
│                  │   2. "profile" (PATH_FILENAME)                         │
│                  │ 接受路径类型的模块将测试这些：                          │
│                  │   lfi_generic   → /api/users/../../etc/passwd/profile   │
│                  │   ssti_detection → /api/users/{{7*7}}/profile           │
│                  │   sqli_error    → /api/users/' OR 1=1--/profile         │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Request          │ 正常运行：                                              │
│                  │   - path_normalization: /api/users/42/../42/profile     │
│                  │   - forbidden_bypass: 如果 403，尝试路径技巧            │
│                  │   - host_header_injection: 测试头反射                   │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Host             │ 如果主机是新的，运行一次。                              │
└──────────────────┴──────────────────────────────────────────────────────────┘
```

### 带嵌套结构的 JSON API

```http
POST /api/graphql HTTP/1.1
Host: example.com
Content-Type: application/json

{"query":"{ user(id: 1) { name } }","variables":{"token":"eyJhbGci..."}}
```

```
┌──────────────────┬──────────────────────────────────────────────────────────┐
│ 范围             │ 行为                                                    │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ InsertionPoint   │ JSON 值被提取为插入点：                                 │
│                  │   1. query = "{ user(id: 1)…"  (JSON_PARAM)            │
│                  │   2. token = "eyJhbGci..."      (JSON_PARAM)           │
│                  │ 当 includeNested=true 时，执行器还会检测到              │
│                  │   "token" 包含 Base64/JWT 值，并在其内部创建嵌套插入点。│
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Request          │   - jwt_vulnerability: 检测请求体中的 JWT，尝试         │
│                  │     算法混淆和弱密钥攻击                                │
│                  │   - xxe_generic: 如果 Content-Type 是 XML，测试 XXE    │
│                  │   - prototype_pollution: 在 JSON 中注入 __proto__      │
├──────────────────┼──────────────────────────────────────────────────────────┤
│ Host             │ graphql_scan 运行一次：内省查询、                       │
│                  │   批量滥用、字段建议枚举                                 │
└──────────────────┴──────────────────────────────────────────────────────────┘
```

---

## 执行顺序

对于每个传入的请求，执行器按以下顺序运行范围：

```
HttpRequestResponse 到达
│
├── 阶段 1: 被动模块（无网络 I/O — 顺序执行）
│   │
│   ├── 1. ScanScopeHost     →  security_headers_missing（每个主机一次）
│   │
│   └── 2. ScanScopeRequest  →  secret_detect, cookie_security_detect,
│                                cors_headers_detect, info_disclosure_detect,
│                                dom_xss_detect, csrf_detect, ...
│
├── 阶段 2: 活跃模块（网络 I/O — 所有三种范围并行运行）
│   │
│   ├── 3a. ScanScopeHost
│   │       cors_misconfiguration, default_credentials, ...
│   │       （仅当主机之前未见过时运行）
│   │
│   ├── 3b. ScanScopeRequest                ─┐
│   │       forbidden_bypass,                 │
│   │       host_header_injection,            ├── 所有三种范围类别
│   │       jwt_vulnerability, ...            │   同时运行
│   │                                         │
│   └── 3c. ScanScopeInsertionPoint          ─┘
│           对于每个参数：
│             sqli_error_based,
│             ssti_detection,
│             lfi_generic, ...
│
└── 阶段 3: 等待所有活跃模块完成
             收集 ResultEvent 漏洞发现
```

被动模块首先运行（顺序执行，因为它们不进行网络 I/O），然后所有三种活跃范围类别同时运行。

---

## 为新模块选择范围

| 问题 | 推荐范围 |
|------|----------|
| 漏洞是否存在于特定参数值中？ | `ScanScopeInsertionPoint` |
| 是否需要修改请求结构（方法、路径、头）？ | `ScanScopeRequest` |
| 是否需要自定义参数迭代逻辑？ | `ScanScopeRequest` |
| 是否是对每个目标主机的一次性检查？ | `ScanScopeHost` |
| 是否需要参数级和请求级测试？ | 使用 `\|` 组合（例如 `ScanScopeRequest \| ScanScopeInsertionPoint`） |

范围在模块构造函数中设置：

```go
func New() *Module {
    return &Module{
        BaseActiveModule: modkit.NewBaseActiveModule(
            ModuleID, ModuleName, ModuleDesc, ModuleShort,
            ModuleConfirmation, ModuleSeverity, ModuleConfidence,
            modkit.ScanScopeInsertionPoint,    // <-- 范围在此处设置
            modkit.AllInsertionPointTypes,
        ),
    }
}
```
# Scanner Modules Reference

Vigolium 内置了 **249 个扫描模块** — 154 个主动模块和 95 个被动模块 — 覆盖 OWASP Top 10 及更广泛的范围。以下列表经过整理；请查阅 `vigolium module ls` 获取实时注册表，因为模块会定期新增。

## 严重性等级

`critical` > `high` > `medium` > `low` > `suspect` > `info`

## 置信度等级

- **certain** — 已确凿确认（载荷已执行，错误已匹配）
- **firm** — 通过行为分析很可能已确认
- **tentative** — 可能但未确认（基于启发式分析）

---

## 主动模块（152 个）

主动模块发送修改后的请求，通过模糊测试、注入和行为分析来检测漏洞。

### XSS

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-xss-light-url-params` | XSS Light - URL Parameters | URL 参数中的反射型 XSS，支持 POST→GET 转换 | High | Firm | `xss`, `injection` |
| `active-xss-light-path` | XSS Light - Path Injection | 通过路径操作的反射型 XSS（递归、截断、追加） | High | Firm | `xss`, `injection` |
| `active-xss-light-param-discovery` | XSS Light - Parameter Discovery | 通过回显参数发现的反射型 XSS | High | Firm | `xss`, `injection` |

### SQL 注入

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-sqli-error-based` | SQLi Error Based | 基于数据库错误信息的报错型 SQL 注入（MySQL、PostgreSQL、MSSQL、Oracle、SQLite） | Critical | Certain | `sqli`, `injection` |
| `active-sqli-boolean-blind` | Blind SQL Injection (Boolean-Based) | 基于 TRUE/FALSE 载荷对的布尔型盲 SQL 注入，三重验证 | High | Certain | `sqli`, `injection` |

### NoSQL 注入

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-nosqli-error-based` | NoSQLi Error Based | 基于错误信息的 NoSQL 注入（MongoDB、CouchDB、Cassandra） | Critical | Certain | `nosqli`, `injection` |
| `active-nosqli-operator-injection` | NoSQL Operator Injection | MongoDB 操作符注入（`$ne`、`$gt`、`$regex`、`$where`），用于认证绕过 | High | Firm | `nosqli`, `injection` |

### 模板注入

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-reflected-ssti` | Reflected SSTI | 通过数学表达式求值的 SSTI（例如 `{{7*7}}=49`） | High | Certain | `ssti`, `injection` |
| `active-ssti-detection` | SSTI Detection | 基于差异分析的 SSTI，采用布尔错误型盲检测技术 | High | Certain | `ssti`, `injection` |
| `active-csti-detection` | Client-Side Template Injection | AngularJS/Vue.js 应用中的 CSTI，通过字面反射检测 | High | Firm | `ssti`, `injection` |

### 文件包含

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-lfi-generic` | LFI Generic | 通过路径遍历载荷的 LFI；匹配已知操作系统文件签名 | Critical | Certain | `lfi`, `injection` |
| `active-lfi-path-traversal` | LFI Path Traversal | 高级 LFI，包含空字节、双重编码、Unicode 绕过 | High | Firm | `lfi`, `injection` |

### 代码执行与注入

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-code-exec` | Code Execution (RCE) | 基于时间盲注的 OS 命令注入（sleep/延迟测量） | Critical | Certain | `rce`, `injection` |
| `active-command-injection-echo` | OS Command Injection (Results-Based) | 带内验证：shell 执行一个唯一的随机算术表达式并回显；两轮 + 基线对比 | Critical | Certain | `rce`, `command-injection`, `injection` |
| `active-command-injection-oast` | OS Command Injection (Out-of-Band) | 通过 nslookup/ping/curl 回调到唯一 OAST 域的盲命令注入 | Critical | Certain | `rce`, `command-injection`, `oast` |
| `active-command-injection-timing` | OS Command Injection (Time-Based) | 延迟缩放盲检测：自适应每目标阈值 + sleep(N) 与 sleep(2N) 在多个独立轮次中的缩放；仅基于时序，故为 Tentative | Critical | Tentative | `rce`, `command-injection`, `injection` |
| `active-crlf-injection` | CRLF Injection | 通过 CR/LF 字符序列在 HTTP 头部中注入 CRLF | Medium | Firm | `injection` |
| `active-xxe-generic` | XXE Generic | 通用 XML 端点中的 XML 外部实体注入 | Critical | Certain | `xxe`, `injection` |
| `active-insecure-deserialization` | Insecure Deserialization | 基于错误检测的 Java、PHP、Python、Ruby 和 .NET 反序列化漏洞 | High | Firm | `injection` |
| `active-input-behavior-probe` | Input Behavior Probe | 通过头部、路径、调试参数和字符探测的行为变化检测 | Info | Tentative | `injection` |

### SSRF 与带外检测（OAST）

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-ssrf-detection` | SSRF Detection | 通过带内探测（内网 IP、云元数据）的 SSRF 检测，基于响应差异分析 | High | Firm | `ssrf`, `injection` |
| `active-oast-probe` | OAST Probe | 通过 DNS/HTTP 回调的盲漏洞检测（盲 SSRF、盲 XXE、盲 RCE） | High | Certain | `ssrf`, `injection` |
| `active-proxy-pingback` | Proxy Pingback | 通过 OAST URL 注入的开放代理/回调端点检测 | High | Certain | `ssrf`, `injection` |

### 配置错误

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-cors-misconfiguration` | CORS Misconfiguration | 宽松的 CORS 策略（反射来源、null 来源、通配符+凭据） | Medium | Firm | `misconfiguration` |
| `active-spring-actuator-misconfig` | Spring Actuator Misconfiguration | 暴露的 Spring Boot Actuator 端点，泄露环境变量、健康信息、配置 | High | Firm | `misconfiguration` |
| `active-host-header-injection` | Host Header Injection | 通过值反射的 Host 头部注入（密码重置/缓存投毒） | Medium | Firm | `misconfiguration` |
| `active-web-cache-poisoning` | Web Cache Poisoning | 通过未键控的头部注入（X-Forwarded-Host、X-Forwarded-Scheme）的缓存投毒 | High | Firm | `misconfiguration` |

### 访问控制

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-forbidden-bypass` | 403/401 Forbidden Bypass | 通过路径操作、头部注入、方法篡改绕过 | Medium | Firm | `auth-bypass` |
| `active-http-method-tampering` | HTTP Method Tampering | 意外启用的 HTTP 方法（PUT、DELETE、PATCH）及覆盖 | Medium | Firm | `auth-bypass` |
| `active-csrf-verify` | CSRF Token Verification | 通过移除、清空或随机化令牌来验证 CSRF 令牌强制实施 | High | Firm | `auth-bypass` |
| `active-idor-detection` | IDOR Detection | 通过邻近 ID 探测的对象 ID 参数授权缺失检测 | High | Tentative | `auth-bypass` |
| `active-mass-assignment` | Mass Assignment | 通过向 JSON API 注入权限键的批量赋值 | High | Firm | `auth-bypass` |
| `active-open-redirect` | Open Redirect | 通过在 Location/meta refresh 中注入外部 URL 的开放重定向 | Medium | Firm | `auth-bypass` |

### 路径分析

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-path-normalization` | Path Normalization | 通过针对中间件/反向代理的遍历载荷检测路径规范化漏洞 | High | Firm | `misconfiguration` |
| `active-nginx-off-by-slash` | Nginx Off-by-Slash | 因缺少尾部斜杠导致的 Nginx 别名遍历 | High | Tentative | `misconfiguration` |
| `active-nginx-path-escape` | Nginx Path Escape Detection | 基于差异分析的别名遍历、URL 编码绕过、分号注入检测 | Info | Tentative | `misconfiguration` |

### 差异与行为检测

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-smart-behavior-detection` | Smart Behavior Detection | 基于差异分析的注入检测，使用 true/false 行为载荷对 | Info | Tentative | `detection` |
| `active-suspect-transform` | Suspect Transform Detection | 表达式求值、引号消耗和 Unicode 规范化检测 | Suspect | Firm | `detection` |
| `active-backslash-transformation` | Backslash Transformation | 转义序列解释、反斜杠消耗、字符处理检测 | Suspect | Firm | `detection` |

### 原型污染

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-prototype-pollution` | Prototype Pollution | 通过 `__proto__` 和 `constructor.prototype` JSON 注入的服务端原型污染 | High | Firm | `javascript`, `injection` |
| `active-client-prototype-pollution` | Client-Side Prototype Pollution | 通过 JavaScript 静态分析（来源 + gadget 模式）的客户端原型污染 | High | Firm | `javascript`, `injection` |

### 竞态条件

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-race-interference` | Race Interference Detection | 通过并行请求分析（输入存储、交叉污染、TOCTOU）的竞态条件检测 | High | Firm | `injection` |

### XML、JWT 与 HTTP 协议

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-xml-saml-security` | XML SAML Security | SAML XML 处理中的 XXE 和 DTD 注入 | High | Firm | `injection` |
| `active-jwt-vulnerability` | JWT Vulnerability | JWT 算法混淆（`none` 算法、空签名、RS256→HS256） | Critical | Certain | `injection` |
| `active-http-request-smuggling` | HTTP Request Smuggling | 通过 Content-Length 与 Transfer-Encoding 冲突的 CL.TE 和 TE.CL 反序列化 | High | Firm | `injection` |

### API 与端点安全

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-graphql-scan` | GraphQL Security Scanner | GraphQL 内省查询、SQL 注入和查询批处理滥用 | Medium | Certain | `api`, `injection` |
| `active-file-upload-scan` | File Upload Scanner | 文件上传绕过（扩展名、空字节、魔数、SVG XXE、HTML XSS） | High | Certain | `injection` |
| `active-default-credentials` | Default Credentials | 使用常见凭据对测试登录端点；支持 CAPTCHA/锁定检测 | High | Certain | `auth-bypass` |
| `active-sensitive-file-discovery` | Sensitive File Discovery | 约 25 个基于标记的敏感文件和约 1,350 个通用路径（.env、.git、日志） | Medium | Tentative | `info-disclosure` |
| `active-jsonp-callback` | JSONP Callback Injection | 通过回调注入的 JSONP 端点，可实现跨域数据窃取 | Medium | Firm | `injection` |

### 代理与工具

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-proxy` | Proxy | 通过已配置代理重放所有请求 | Info | Firm | `utility`, `light` |
| `active-proxy-header-trust` | Proxy Header Trust | 通过 X-Forwarded-* 操作的跨框架代理头部信任问题 | High | Firm | `misconfiguration`, `moderate` |
| `active-api-rate-limit-bypass` | API Rate Limit Bypass | 通过 IP 欺骗头部的速率限制绕过 | Medium | Firm | `auth-bypass`, `moderate` |
| `active-websocket-security` | WebSocket Security | 不安全的 WebSocket 升级策略和缺失的来源验证 | High | Firm | `misconfiguration`, `light` |
| `active-swagger-disclose` | Swagger Disclosure | 暴露的 Swagger/OpenAPI 文档 | Medium | Firm | `api`, `info-disclosure`, `light` |
| `active-backup-file-discovery` | Backup File Discovery | 从主机名和年份变体推导出的暴露备份归档 | Medium | Tentative | `sensitive-file`, `moderate` |
| `active-angular-template-injection` | Angular Template Injection | 通过表达式求值的 Angular 模板注入 | High | Firm | `angular`, `injection`, `ssti` |

### SQL 注入（基于时间）

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-sqli-time-based-header` | SQLi Time Based - Header | HTTP 头部中的基于时间 SQL 注入 | Critical | Certain | `injection`, `sqli`, `heavy` |
| `active-sqli-time-based-params` | SQLi Time Based - Params | 参数中的基于时间 SQL 注入 | Critical | Certain | `injection`, `sqli`, `heavy` |
| `active-sqli-time-blind` | Blind SQL Injection (Time-Based) | 基于时间的盲 SQL 注入 | High | Firm | `injection`, `sqli`, `heavy` |

### SSRF 与 SSTI（盲注）

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-ssrf-blind` | Blind SSRF Detection | 通过 OAST 回调的盲 SSRF 检测 | High | Firm | `ssrf`, `injection`, `heavy` |
| `active-ssti-blind` | Blind SSTI | 通过 OAST 回调和时间延迟载荷的盲 SSTI | Critical | Firm | `injection`, `ssti`, `heavy` |

### 框架安全

#### Next.js

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-nextjs-data-leakage` | Next.js Data Route Leakage | 未授权访问 `/_next/data/<buildId>/<path>.json` | High | Firm | `nextjs`, `javascript` |
| `active-nextjs-middleware-bypass` | Next.js Middleware Bypass | CVE-2025-29927 及路径规范化绕过 | Critical | Firm | `nextjs`, `javascript` |
| `active-nextjs-image-ssrf` | Next.js Image Optimizer SSRF | 通过 `/_next/image` 的 SSRF，使用 OAST 和带内探测 | High | Firm | `nextjs`, `javascript` |
| `active-nextjs-draft-mode-exposure` | Next.js Draft Mode Exposure | 不安全或未受保护的 Draft/Preview Mode 端点 | High | Firm | `nextjs`, `javascript` |
| `nextjs-version-audit` | Next.js Version Audit | 识别 Next.js 版本并映射到已知 CVE 公告 | High | Firm | `nextjs`, `javascript`, `fingerprint` |
| `active-js-devserver-exposure` | JS Dev Server Exposure | 暴露的 webpack HMR、Vite、Nuxt、Remix 开发服务器端点 | Medium | Firm | `javascript` |

#### Spring / Java

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-spring-actuator-misconfig` | Spring Actuator Misconfiguration | 暴露的 Spring Boot Actuator 端点 | High | Firm | `spring`, `java` |
| `active-spring-boot-admin-exposure` | Spring Boot Admin Exposure | 暴露的 Spring Boot Admin 仪表板 | High | Firm | `spring`, `java` |
| `active-spring-cloud-config-exposure` | Spring Cloud Config Exposure | 暴露的 Config Server 端点泄露密钥 | Critical | Firm | `spring`, `java` |
| `active-spring-data-rest-exposure` | Spring Data REST Exposure | 自动暴露的带有 HAL/HATEOAS 的仓库端点 | Medium | Firm | `spring`, `java` |
| `active-spring-debug-exposure` | Spring Debug Exposure | 调试端点、Whitelabel 错误页面、堆栈跟踪 | Medium | Firm | `spring`, `java` |
| `active-spring-gateway-exposure` | Spring Gateway Exposure | 暴露的 Cloud Gateway Actuator 泄露路由信息 | High | Firm | `spring`, `java` |
| `active-spring-h2-console-exposure` | Spring H2 Console Exposure | 暴露的 H2 数据库 Web 控制台 | Critical | Firm | `spring`, `java`, `rce` |
| `active-spring-jolokia-exposure` | Spring Jolokia Exposure | 暴露的 Jolokia JMX 端点 | High | Firm | `spring`, `java` |
| `active-java-appserver-console` | Java App Server Console | 暴露的管理控制台（WildFly、WebLogic、GlassFish） | High | Firm | `java`, `tomcat` |
| `active-java-sensitive-files` | Java Sensitive Files | Java 配置文件、WEB-INF、META-INF、构建产物 | Medium | Tentative | `java`, `sensitive-file` |
| `active-tomcat-manager-exposure` | Tomcat Manager Exposure | 暴露的 Tomcat Manager 和 Host Manager 接口 | High | Firm | `tomcat`, `java` |

#### Django / Flask / FastAPI (Python)

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `django-admin-exposure` | Django Admin Exposure | 暴露的 Django 管理面板和登录页面 | Medium | Firm | `django`, `python` |
| `django-browsable-api-exposure` | Django Browsable API Exposure | 通过 Accept 头部检测到的 DRF 可浏览 API | Low | Firm | `django`, `python` |
| `django-debug-exposure` | Django Debug Exposure | Django DEBUG=True 信息泄露 | High | Firm | `django`, `python` |
| `django-debug-toolbar-exposure` | Django Debug Toolbar Exposure | 暴露的 django-debug-toolbar 面板 | High | Firm | `django`, `python` |
| `flask-werkzeug-debugger` | Flask Werkzeug Debugger | 暴露的 Werkzeug 交互式调试器（RCE） | Critical | Certain | `flask`, `python`, `rce` |
| `fastapi-docs-exposure` | FastAPI Docs Exposure | 暴露的 FastAPI 交互式 API 文档 | Low | Firm | `fastapi`, `python` |
| `fastapi-auth-inconsistency` | FastAPI Auth Inconsistency | 通过 OpenAPI 模式发现未受保护的操作 | Medium | Firm | `fastapi`, `python` |

#### Laravel / Symfony / PHP

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-laravel-admin-exposure` | Laravel Admin Exposure | 暴露的管理面板、API 文档、GraphQL 端点 | High | Firm | `laravel`, `php` |
| `active-laravel-devtool-exposure` | Laravel Developer Tool Exposure | 暴露的 Web Tinker、Clockwork、Pulse、日志查看器 | High | Firm | `laravel`, `php` |
| `active-laravel-ignition-rce` | Laravel Ignition RCE | CVE-2021-3129 RCE，通过暴露的 Ignition 端点 | Critical | Firm | `laravel`, `php`, `rce` |
| `active-laravel-misconfig` | Laravel Misconfiguration | 调试模式、暴露的调试栏、应用日志 | High | Firm | `laravel`, `php` |
| `active-laravel-sensitive-files` | Laravel Sensitive Files | PHPUnit 配置、SQLite 数据库、存储内部文件 | Medium | Tentative | `laravel`, `php` |
| `active-symfony-misconfig` | Symfony Misconfiguration | 暴露的分析器、调试工具栏、开发前端控制器 | High | Firm | `symfony`, `php` |
| `active-php-composer-exposure` | PHP Composer Exposure | 暴露的 Composer 清单、vendor 目录 | High | Firm | `php` |
| `active-php-debug-exposure` | PHP Debug Exposure | 暴露的 phpinfo、PHP-FPM 状态、phpMyAdmin | Medium | Firm | `php` |
| `active-php-framework-debug` | PHP Framework Debug Exposure | Yii、CodeIgniter、CakePHP 的调试端点 | Medium | Firm | `php` |
| `active-php-path-info-misconfig` | PHP PATH_INFO Misconfiguration | cgi.fix_pathinfo 路由歧义 | Medium | Firm | `php` |
| `active-php-source-disclosure` | PHP Source Disclosure | 通过 .phps 处理器的 PHP 源代码泄露 | High | Firm | `php` |

#### Rails (Ruby)

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-rails-info-exposure` | Rails Info Exposure | 生产环境中暴露的 Rails 开发/调试端点 | High | Firm | `rails`, `ruby` |
| `active-rails-admin-dashboard` | Rails Admin Dashboard | 暴露的 Rails 生态系统管理面板 | High | Firm | `rails`, `ruby` |
| `active-rails-sensitive-files` | Rails Sensitive Files | 暴露的 Rails 配置、凭据、构建产物 | Medium | Tentative | `rails`, `ruby` |
| `active-rails-action-mailbox-probe` | Rails Action Mailbox Probe | 暴露的 Action Mailbox 入口端点 | Medium | Firm | `rails`, `ruby` |
| `active-rails-active-storage-probe` | Rails Active Storage Probe | 暴露的 Active Storage 直接上传端点 | Medium | Firm | `rails`, `ruby` |

#### Express (Node.js)

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-express-debug-probe` | Express Debug Probe | 堆栈跟踪和调试信息泄露 | Low | Firm | `express`, `javascript` |
| `active-express-directory-listing` | Express Directory Listing | 通过 serve-index 中间件的目录列表 | Low | Firm | `express`, `javascript` |
| `active-express-trust-proxy-misconfig` | Express Trust Proxy Misconfiguration | 通过 X-Forwarded-* 的信任代理配置错误 | Medium | Firm | `express`, `javascript` |

#### ASP.NET / IIS

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-aspnet-blazor-exposure` | ASP.NET Blazor Exposure | 暴露的 Blazor WebAssembly 程序集和 Server 端点 | Medium | Firm | `aspnet` |
| `active-aspnet-health-exposure` | ASP.NET Health Endpoint Exposure | 暴露的健康检查、监控仪表板、指标 | Medium | Firm | `aspnet` |
| `active-aspnet-identity-probe` | ASP.NET Identity Probe | 暴露的 Identity 端点和 IdentityServer | Medium | Firm | `aspnet` |
| `active-aspnet-misconfig` | ASP.NET Misconfiguration | 暴露的诊断、调试端点、详细错误信息 | High | Firm | `aspnet` |
| `active-aspnet-sensitive-files` | ASP.NET Sensitive Files | 暴露的配置文件、备份、敏感目录 | Medium | Tentative | `aspnet` |
| `active-aspnet-service-exposure` | ASP.NET Service Exposure | 暴露的 ASMX、WCF、OData、遗留服务路径 | Medium | Firm | `aspnet` |
| `active-aspnet-viewstate-scan` | ASP.NET ViewState Scan | ViewState MAC 禁用、事件验证绕过 | High | Firm | `aspnet` |
| `active-iis-shortname-discovery` | IIS Short Filename Discovery | 通过波浪号 oracle 的 IIS 8.3 短文件名枚举 | Medium | Certain | `aspnet` |

#### Firebase

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-firebase-auth-misconfig` | Firebase Auth Misconfiguration | Firebase Authentication 配置错误 | Medium | Firm | `firebase` |
| `active-firebase-functions-exposure` | Firebase Functions Exposure | 未认证的 Cloud Functions | High | Firm | `firebase` |
| `active-firebase-misconfig` | Firebase Misconfiguration | 暴露的 Firebase 配置、安全规则、凭据 | High | Firm | `firebase` |
| `active-firebase-rtdb-exposure` | Firebase RTDB Exposure | 公开可读的 Realtime Database | Critical | Certain | `firebase` |
| `active-firebase-storage-exposure` | Firebase Storage Exposure | 公开可访问的 Cloud Storage 存储桶 | High | Certain | `firebase`, `cloud` |

#### 云基础设施

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-cloud-bucket-takeover` | Cloud Bucket Takeover | 易受接管攻击的悬空云存储桶 | High | Firm | `cloud` |
| `active-cloud-origin-bypass` | Cloud Origin Bypass | 绕过 CDN 安全直接访问源站 | Medium | Firm | `cloud` |
| `active-cloud-public-read` | Cloud Public Read | 云存储上公开可读的敏感路径 | High | Firm | `cloud` |
| `active-cloud-storage-listing` | Cloud Storage Listing | 公开可列举的 S3 存储桶和 Azure 容器 | High | Certain | `cloud` |

#### CMS（WordPress、Drupal、Joomla、Magento）

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `active-wp-misconfig` | WordPress Misconfiguration | 暴露的配置文件、调试日志、危险端点 | High | Firm | `wordpress`, `php` |
| `active-wp-user-enum` | WordPress User Enumeration | 通过作者归档和 REST API 的用户枚举 | Medium | Certain | `wordpress`, `php` |
| `active-wp-xmlrpc` | WordPress XML-RPC Abuse | XML-RPC 多重调用暴力破解和 pingback 滥用 | Medium | Firm | `wordpress`, `php` |
| `active-wp-ajax-exposure` | WordPress AJAX Action Exposure | 插件中公开可访问的 AJAX 操作 | High | Firm | `wordpress`, `php` |
| `active-drupal-misconfig` | Drupal Misconfiguration | 暴露的配置文件、更新脚本、安装程序 | High | Firm | `drupal`, `php` |
| `active-drupal-user-enum` | Drupal User Enumeration | 通过用户资料和 JSON:API 的用户枚举 | Medium | Certain | `drupal`, `php` |
| `active-joomla-misconfig` | Joomla Misconfiguration | 暴露的配置备份、日志/临时目录、调试设置 | High | Firm | `joomla`, `php` |
| `active-joomla-user-enum` | Joomla User Enumeration | 通过注册、API、管理员登录的用户枚举 | Medium | Firm | `joomla`, `php` |
| `active-magento-misconfig` | Magento Misconfiguration | 暴露的安装向导、下载器、版本文件 | High | Firm | `magento`, `php` |
| `active-cms-installer-exposure` | CMS Installer Exposure | 暴露的 WordPress、Drupal 和 Joomla 安装向导 | Critical | Firm | `wordpress`, `drupal`, `joomla` |

---

## 被动模块（91 个）

被动模块分析现有的请求/响应对，不发送新流量。

### XSS

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-dom-xss-detect` | DOM XSS Detect | DOM XSS 来源到接收器的数据流（location.hash、innerHTML、eval、document.write） | Medium | Firm | `xss` |

### 身份验证与会话

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-auth-headers-detect` | Auth Headers Detect | 请求中的 Authorization 头部（Bearer 令牌、API 密钥） | High | Firm | `session`, `auth` |
| `passive-jwt-weak-secret` | JWT Weak Secret Detection | 针对约 104K 单词列表的 JWT HMAC 密钥离线暴力破解 | High | Firm | `session`, `auth` |
| `passive-cookie-security-detect` | Cookie Security Detect | 不安全的 Cookie 属性（缺少 Secure、HttpOnly、SameSite） | Low | Certain | `session`, `auth` |
| `passive-password-autocomplete-detect` | Password Autocomplete | 没有 `autocomplete="off"` 的密码字段 | Info | Certain | `session`, `auth` |

### 注入信号

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-sql-syntax-detect` | SQL Syntax in Request | HTTP 请求参数值中的 SQL 语句/关键字 | Info | Firm | `injection` |
| `passive-serialized-object-detect` | Serialized Object Detection | 请求参数中的序列化 Java/PHP/.NET/Python 对象 | Medium | Firm | `injection` |
| `passive-input-reflection-detect` | Input Reflection Detect | 请求参数值在响应体中逐字反射 | Info | Tentative | `injection` |
| `passive-base64-data-detect` | Base64 Data Detect | 请求/响应中感兴趣的 Base64 数据（JSON、PHP 对象、URL、Java 对象） | Info | Tentative | `injection` |

### 信息泄露

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-secret-detect` | Secret Detection | 通过 Kingfisher 引擎检测泄露的密钥、API 密钥和凭据 | High | Firm | `info-disclosure` |
| `passive-info-disclosure-detect` | Info Disclosure Detect | 服务器版本、内网 IP、堆栈跟踪、调试信息 | Low | Firm | `info-disclosure` |
| `passive-error-message-detect` | Error Message Detect | 来自调试页面、Apache、ASP.NET、Java、PHP、Ruby、Node.js、SQL 的错误信息 | Info | Firm | `info-disclosure` |
| `passive-sourcemap-detect` | Sourcemap Exposure | 通过 SourceMappingURL 引用暴露的 JavaScript sourcemap | Low | Firm | `info-disclosure` |
| `passive-sensitive-url-params` | Sensitive URL Params | 在 URL 查询参数中传递的密码、令牌、API 密钥 | Medium | Firm | `info-disclosure` |
| `passive-content-type-mismatch` | Content Type Mismatch | Content-Type/主体不匹配，可能导致 MIME 混淆攻击 | Low | Firm | `info-disclosure` |

### 安全头部与配置

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-security-headers-missing` | Security Headers Missing | 缺少 X-Content-Type-Options、X-Frame-Options、HSTS、CSP、Permissions-Policy；缺少/弱 Referrer-Policy；可缓存的敏感 HTTPS 响应 | Info | Certain | `header-security` |
| `passive-mixed-content-detect` | Mixed Content Detect | HTTPS 页面上加载的 HTTP 资源（src、href、action 属性） | Low | Certain | `header-security` |

### CORS 与重定向

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-cors-headers-detect` | CORS Headers Detect | 宽松的 CORS 头部（通配符来源、启用凭据） | Low | Firm | `cors` |
| `passive-openredirect-params` | Open Redirect Params | 与开放重定向相关的 URL 参数名（redirect、url、next、goto） | Info | Tentative | `cors` |
| `passive-oauth-facebook-detect` | Facebook OAuth Detect | 用于 OAuth 流程分析的 Facebook OAuth 重定向参数 | Medium | Firm | `cors` |

### 访问控制

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-csrf-detect` | CSRF Detection | 缺少反 CSRF 保护的状态变更请求（POST/PUT/DELETE/PATCH） | Medium | Tentative | `auth-bypass` |
| `passive-idor-params-detect` | IDOR Parameter Detection | 引用对象标识符的参数，用于 IDOR/BOLA 分类 | Info | Tentative | `auth-bypass` |

### 密码学

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-crypto-weakness-detect` | Cryptographic Weakness | PHP 魔术哈希、弱 MD5/SHA1、填充 oracle 错误、未受保护的加密 Cookie | Medium | Tentative | `crypto` |

### 异常检测

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-anomaly-ranking` | Anomaly Ranking | 跨每主机响应批次的统计异常检测；更新 risk_score | Suspect | Tentative | `detection` |

### JS 框架安全（运行时分析）

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-js-framework-fingerprint` | JS Framework Fingerprint | 识别 Next.js、Nuxt、Angular、React、Remix、SvelteKit、Gatsby；提取 buildId | Info | Certain | `javascript` |
| `passive-ssr-data-exposure` | SSR Data Exposure | SSR 状态 blob 中的敏感数据（`__NEXT_DATA__`、`__NUXT__`、`__INITIAL_STATE__`） | Medium | Firm | `javascript` |
| `passive-cache-auth-misconfiguration` | Cache-Auth Misconfiguration | 缺少 Vary 头部的包含用户特定数据的可缓存响应 | Medium | Firm | `javascript` |
| `passive-server-action-auth` | Server Action Auth Check | 执行变更操作但缺少授权的 Next.js Server Actions | High | Tentative | `javascript` |
| `passive-nextjs-config-audit` | Next.js Config Audit | 不安全的 Next.js 配置（dangerouslyAllowSVG、通配符图片域名、生产环境 sourcemap） | Medium | Firm | `javascript` |
| `passive-client-auth-guard` | Client Auth Guard Check | 仅客户端认证守卫（useEffect 重定向）缺少服务端强制实施 | High | Tentative | `javascript` |
| `passive-cache-data-leak` | Cache Data Leak | 带认证的 `getStaticProps`/force-static、没有用户作用域键的 `unstable_cache` | Medium | Tentative | `javascript` |

### JS 框架安全（源码分析）

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-unsafe-html-sink` | Unsafe HTML Sink | 原始 HTML 注入接收器：`dangerouslySetInnerHTML`、`v-html`、`{@html}`、`innerHTML` | High | Firm | `javascript` |
| `passive-insecure-token-storage` | Insecure Token Storage | 存储在 `localStorage`/`sessionStorage` 中的认证令牌 | Medium | Firm | `javascript` |
| `passive-env-secret-exposure` | Environment Secret Exposure | `NEXT_PUBLIC_`、`VITE_`、`REACT_APP_` 公共环境变量中的密钥；提供服务的 `.env` 文件 | High | Firm | `javascript` |
| `passive-build-misconfig-detect` | Build Misconfiguration | 生产环境 sourcemap、生产环境中的开发模式、SVG XSS 风险、宽泛的图片 `remotePatterns` | High | Firm | `javascript` |

### 框架指纹识别

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-aspnet-fingerprint` | ASP.NET Fingerprint | 识别 ASP.NET 版本和配置 | Info | Firm | `aspnet`, `fingerprint` |
| `passive-aspnet-viewstate-detect` | ASP.NET ViewState Detect | 分析 ViewState 字段的安全问题 | Medium | Firm | `aspnet` |
| `passive-django-fingerprint` | Django Fingerprint | 识别 Django 框架指示信息 | Info | Firm | `django`, `python`, `fingerprint` |
| `passive-express-fingerprint` | Express Fingerprint | 识别 Express.js 指示信息 | Info | Firm | `express`, `fingerprint` |
| `passive-fastapi-fingerprint` | FastAPI Fingerprint | 识别 FastAPI 框架指示信息 | Info | Firm | `fastapi`, `python`, `fingerprint` |
| `passive-firebase-fingerprint` | Firebase Fingerprint | 识别 Firebase SDK 使用情况和配置 | Info | Firm | `firebase`, `fingerprint` |
| `passive-flask-fingerprint` | Flask Fingerprint | 识别 Flask 框架指示信息 | Info | Firm | `flask`, `python`, `fingerprint` |
| `passive-laravel-fingerprint` | Laravel Fingerprint | 识别 Laravel 框架指示信息 | Info | Firm | `laravel`, `php`, `fingerprint` |
| `passive-rails-fingerprint` | Rails Fingerprint | 识别 Rails 框架指示信息 | Info | Firm | `rails`, `ruby`, `fingerprint` |
| `passive-spring-fingerprint` | Spring Fingerprint | 识别 Spring Boot 指示信息 | Info | Firm | `spring`, `java`, `fingerprint` |
| `passive-drupal-fingerprint` | Drupal Fingerprint | 识别 Drupal CMS 指示信息 | Info | Firm | `drupal`, `php`, `fingerprint` |
| `passive-joomla-fingerprint` | Joomla Fingerprint | 识别 Joomla CMS 指示信息 | Info | Firm | `joomla`, `php`, `fingerprint` |
| `passive-wp-fingerprint` | WordPress Fingerprint | 识别 WordPress CMS 指示信息 | Info | Firm | `wordpress`, `php`, `fingerprint` |

### API 与协议分析

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-api-version-detect` | API Version Detection | 检测 URL 和头部中的 API 版本模式 | Info | Firm | `api` |
| `passive-graphql-introspection-detect` | GraphQL Introspection Detect | 检测已启用的 GraphQL 内省查询 | Medium | Certain | `api`, `graphql` |
| `passive-grpc-web-detect` | gRPC-Web Detect | 检测 gRPC-Web 流量模式 | Info | Firm | `api` |
| `passive-endpoint-classifier` | Endpoint Classifier | 对端点类型进行分类（API、auth、admin、static） | Info | Tentative | `api` |

### 安全头部与策略

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-csp-weakness-audit` | CSP Weakness Audit | Content-Security-Policy 弱点和绕过 | Medium | Firm | `header-security` |
| `passive-permissions-policy-detect` | Permissions-Policy Detect | 缺失或弱的 Permissions-Policy/Feature-Policy | Info | Certain | `header-security` |
| `passive-hsts-preload-audit` | HSTS Preload Audit | HSTS 头部配置和预加载就绪状态 | Info | Firm | `header-security` |
| `passive-subresource-integrity-detect` | Subresource Integrity Detect | 缺少 SRI 属性的脚本/样式 | Low | Firm | `header-security` |
| `passive-cors-vary-origin-missing` | CORS Vary: Origin Missing | 缺少 Vary: Origin 头部的 CORS 响应 | Low | Firm | `cors`, `header-security` |

### 云与 Firebase

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-cloud-storage-fingerprint` | Cloud Storage Fingerprint | 从 URL/头部识别云存储提供商 | Info | Firm | `cloud`, `fingerprint` |
| `passive-cloud-storage-error-info` | Cloud Storage Error Info | 泄露存储桶名称的云存储错误信息 | Low | Firm | `cloud`, `info-disclosure` |
| `passive-cloud-signed-url-leak` | Cloud Signed URL Leak | 权限过大或过期时间过长的云签名 URL | Medium | Firm | `cloud`, `info-disclosure` |

### CMS 检测

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-drupal-api-detect` | Drupal API Detect | 检测 Drupal JSON:API 和 REST 端点 | Info | Firm | `drupal`, `api` |
| `passive-joomla-api-detect` | Joomla API Detect | 检测 Joomla API 端点和版本 | Info | Firm | `joomla`, `api` |
| `passive-wp-rest-api-detect` | WordPress REST API Detect | 检测 WordPress REST API 端点 | Info | Firm | `wordpress`, `api` |

### 高级 JS 框架分析

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-nextjs-dynamic-param-audit` | Next.js Dynamic Param Audit | 审计 Next.js 动态路由参数的注入风险 | Medium | Tentative | `nextjs`, `javascript` |
| `passive-nextauth-config-audit` | NextAuth.js Config Audit | 审计 NextAuth.js 配置的安全问题 | Medium | Firm | `nextjs`, `javascript` |
| `passive-nuxt-config-audit` | Nuxt Config Audit | 审计 Nuxt.js 配置的安全问题 | Medium | Firm | `nuxt`, `javascript` |
| `passive-remix-loader-exposure` | Remix Loader Exposure | 检测暴露的 Remix loader 数据 | Medium | Firm | `remix`, `javascript` |
| `passive-ssr-hydration-xss` | SSR Hydration XSS | 检测通过 SSR 水合不匹配导致的 XSS | High | Firm | `javascript`, `xss` |
| `passive-server-action-bind-audit` | Server Action Bind Audit | 审计 Next.js Server Action .bind() 使用的安全性 | Medium | Tentative | `nextjs`, `javascript` |
| `passive-server-action-input-audit` | Server Action Input Audit | 审计 Next.js Server Action 输入验证 | Medium | Tentative | `nextjs`, `javascript` |
| `passive-server-only-boundary-audit` | Server-Only Boundary Audit | 审计仅服务端模块边界强制实施 | Medium | Tentative | `nextjs`, `javascript` |
| `passive-javascript-uri-sink` | JavaScript URI Sink | 检测链接和事件处理器中的 javascript: URI 使用 | High | Firm | `javascript`, `xss` |
| `passive-wasm-module-detect` | WebAssembly Module Detect | 检测 WebAssembly 模块加载 | Info | Firm | `javascript` |

### 会话与身份验证（被动）

| 模块 ID | 名称 | 描述 | 严重性 | 置信度 | 标签 |
|---|---|---|---|---|---|
| `passive-express-session-audit` | Express Session Audit | 审计 Express 会话 Cookie 配置 | Medium | Firm | `express`, `session` |
| `passive-jwt-claims-detect` | JWT Claims Detect | 分析 JWT 载荷声明中的安全问题 | Info | Firm | `auth`, `session` |
| `passive-jackson-deserialize-detect` | Jackson Deserialization Detect | 检测 Jackson 默认类型指示信息 | Medium | Firm | `java`, `injection` |
| `passive-python-debug-detect` | Python Debug Detect | 检测 Python 调试/回溯指示信息 | Low | Firm | `python` |
| `passive-rails-debug-detect` | Rails Debug Detect | 检测 Rails 调试页面指示信息 | Medium | Firm | `rails`, `ruby` |
| `passive-rails-action-cable-detect` | Rails Action Cable Detect | 检测 Rails Action Cable WebSocket 端点 | Info | Firm | `rails`, `ruby` |
| `passive-rails-active-storage-detect` | Rails Active Storage Detect | 检测 Active Storage blob URL 和签名令牌 | Info | Firm | `rails`, `ruby` |
| `passive-sensitive-api-fields-detect` | Sensitive API Fields Detect | 检测 API 响应中的敏感字段名 | Medium | Tentative | `api`, `info-disclosure` |

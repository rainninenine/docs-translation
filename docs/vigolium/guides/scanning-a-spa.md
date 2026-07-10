# Scanning a Single Page Application (SPA)

单页应用（SPA）对安全扫描器提出了独特的挑战。与返回完整 HTML 页面的传统服务器渲染应用不同，SPA 使用 JavaScript 动态加载内容和路由。Vigolium 通过浏览器爬虫 Spitolas 和高级 JavaScript 分析来应对这些挑战。

## 问题

传统扫描器通过以下方式发现端点：
1. 爬取 HTML 链接（`<a href>`）
2. 解析 `sitemap.xml`
3. 暴力破解目录

SPA 打破了这些假设：
- 路由由 JavaScript 处理（例如 `react-router`、`vue-router`）
- 内容通过 XHR/fetch API 调用动态加载
- 初始 HTML 通常是一个空的 `<div id="root">`

## 解决方案：浏览器爬虫

Vigolium 的 Spitolas 爬虫驱动一个真实的 Chromium 浏览器来发现 SPA 路由：

```bash
# 启用浏览器爬虫扫描 SPA
vigolium scan -t https://spa.example.com --spider

# 更深入的 SPA 覆盖
vigolium scan -t https://spa.example.com --spider --spider-strategy deep

# 限制爬取深度
vigolium scan -t https://spa.example.com --spider --spider-max-depth 5
```

### Spitolas 如何工作

1. **启动浏览器** — 启动一个无头 Chromium 实例
2. **捕获初始状态** — 对初始 DOM 进行快照
3. **与页面交互** — 点击按钮、填写表单、遍历 iframe
4. **跟踪状态变化** — 检测 DOM 变化以发现新路由
5. **捕获网络流量** — 记录所有 XHR/fetch 调用及其响应
6. **重复** — 继续探索直到覆盖所有状态

## 无需浏览器的 SPA 扫描

如果您无法使用浏览器爬虫（例如在 CI/CD 环境中），Vigolium 仍然可以通过以下方式有效扫描 SPA：

### 1. 提供路由映射

```bash
# 从源代码提取路由并作为端点提供
vigolium scan -t https://spa.example.com \
  --endpoints routes.json
```

`routes.json` 格式：
```json
[
  "https://spa.example.com/",
  "https://spa.example.com/login",
  "https://spa.example.com/dashboard",
  "https://spa.example.com/users",
  "https://spa.example.com/users/1",
  "https://spa.example.com/settings"
]
```

### 2. 使用 API 规范

SPA 通常依赖后端 API。提供 API 规范可以让 Vigolium 发现后端端点：

```bash
vigolium scan -I openapi -i api-spec.yaml -t https://api.example.com
```

### 3. 结合两者

```bash
# 爬取 SPA 前端 + 扫描后端 API
vigolium scan -t https://spa.example.com \
  --spider \
  -I openapi -i api-spec.yaml -t https://api.example.com
```

## 配置 Spitolas

```yaml
# vigolium-configs.yaml
spidering:
  max_states: 100          # 发现的最大状态数
  max_depth: 5             # 最大爬取深度
  headless: true           # 无头模式
  strategy: adaptive       # 或 "" 使用默认（BFS/DFS）
  browser_count: 1         # 并行浏览器实例数
  max_duration: 10m        # 最大爬取时长
```

## 提示

- **从主页开始** — 始终从 SPA 的入口 URL 开始爬取
- **提供凭据** — 如果 SPA 需要登录，使用预钩子注入认证标头
- **限制范围** — 使用范围规则将爬虫限制在相关路径
- **检查 API 流量** — 使用 `vigolium traffic` 查看爬虫发现了哪些 API 调用
- **使用自适应策略** — `adaptive` 策略使用多臂赌博机算法优先探索最有前景的状态

## 延伸阅读

- [Spitolas — Browser-Based Web Crawler](../native-scan/phases/spidering.md) — 爬虫的详细文档
- [Native Scan Phases](native-scan-phases.md) — 扫描阶段概览
- [Scanning an API](scanning-an-api.md) — API 扫描指南

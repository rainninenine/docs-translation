# Transparent Proxy

Vigolium 服务器包含一个内置的透明 HTTP 代理，用于零配置的流量捕获。这对于捕获来自无法直接修改的工具（如 `curl`、`httpx`、`nuclei` 或其他安全工具）的流量非常有用。

## 启动代理

```bash
vigolium server --ingest-proxy-port 9003
```

这会在端口 9003 上打开一个记录型 HTTP 代理。通过它路由的纯 HTTP 流量会被捕获到数据库中。

## 使用代理

### 使用 curl

```bash
curl -x http://localhost:9003 http://example.com/api/users
```

### 使用 httpx

```bash
httpx -l urls.txt -proxy http://localhost:9003
```

### 使用 nuclei

```bash
nuclei -l urls.txt -proxy-url http://localhost:9003
```

### 使用浏览器

在浏览器中配置代理设置指向 `localhost:9003`。所有 HTTP 流量都会被捕获。HTTPS `CONNECT` 隧道会**透传**而不记录。

## 工作原理

1. 代理接收 HTTP 请求
2. 将请求转发到目标服务器
3. 将请求和响应都记录到数据库中
4. 将响应返回给客户端

HTTPS 流量通过 `CONNECT` 隧道透传——代理不会解密或记录 HTTPS 流量。

## 与扫描集成

一旦流量被捕获，您就可以对其运行扫描：

```bash
# 扫描所有捕获的流量
vigolium scan --only dynamic-assessment

# 或使用服务器 API
curl -X POST http://localhost:9002/api/scans/run
```

## 配置

```yaml
# vigolium-configs.yaml
server:
  ingest_proxy_port: 9003    # 代理端口（0 = 禁用）
  ingest_proxy_host: "0.0.0.0"  # 绑定地址
```

## 注意事项

- 仅记录 HTTP 流量
- HTTPS 流量透传而不记录
- 代理是透明的——它不修改请求或响应
- 代理不缓存——它只是记录并转发
- 代理适用于临时流量捕获，不建议用于生产级代理需求

## 延伸阅读

- [Running the Server](running-the-server.md) — 服务器启动和配置
- [Ingestion](ingestion.md) — 其他流量导入方式
- [Server & API Architecture](../architecture/server-and-api.md) — 服务器架构详情

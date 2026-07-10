---
title: "LLM API"
description: "兼容 OpenAI 的聊天补全和嵌入"
---

# LLM API

无需执行工作流即可直接访问大语言模型功能。

## 聊天补全

向配置的 LLM 供应商发送聊天补全请求。

```bash
curl -X POST http://localhost:8002/osm/api/llm/v1/chat/completions \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a security analyst."},
      {"role": "user", "content": "Analyze the security posture of example.com"}
    ],
    "max_tokens": 1000,
    "temperature": 0.7
  }'
```

**请求体：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `messages` | array | 是 | 包含 `role` 和 `content` 的消息对象数组 |
| `model` | string | 否 | 使用的模型（默认为供应商的默认模型） |
| `max_tokens` | int | 否 | 响应中的最大令牌数 |
| `temperature` | float | 否 | 采样温度（0.0-2.0） |
| `top_p` | float | 否 | Top-p 采样参数 |
| `top_k` | int | 否 | Top-k 采样参数 |
| `n` | int | 否 | 生成的补全数量 |
| `stream` | bool | 否 | 启用流式传输（暂不支持） |
| `tools` | array | 否 | 用于函数调用的工具定义 |
| `tool_choice` | string/object | 否 | 工具选择策略 |
| `response_format` | object | 否 | 响应格式（`{"type": "json_object"}`） |

**消息角色：**
- `system` - 设置助手行为的系统提示
- `user` - 用户消息
- `assistant` - 之前的助手响应
- `tool` - 工具调用结果

**响应：**
```json
{
  "id": "chatcmpl-abc123",
  "model": "gpt-4",
  "content": "Based on my analysis of example.com...",
  "finish_reason": "stop",
  "usage": {
    "prompt_tokens": 50,
    "completion_tokens": 200,
    "total_tokens": 250
  }
}
```

---

## 带工具（函数调用）

```bash
curl -X POST http://localhost:8002/osm/api/llm/v1/chat/completions \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "What DNS records exist for example.com?"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "dns_lookup",
          "description": "Look up DNS records for a domain",
          "parameters": {
            "type": "object",
            "properties": {
              "domain": {"type": "string", "description": "Domain to look up"},
              "record_type": {"type": "string", "enum": ["A", "AAAA", "MX", "TXT", "NS"]}
            },
            "required": ["domain"]
          }
        }
      }
    ],
    "tool_choice": "auto"
  }'
```

**带工具调用的响应：**
```json
{
  "id": "chatcmpl-xyz789",
  "model": "gpt-4",
  "content": null,
  "finish_reason": "tool_calls",
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "dns_lookup",
        "arguments": "{\"domain\": \"example.com\", \"record_type\": \"A\"}"
      }
    }
  ],
  "usage": {
    "prompt_tokens": 100,
    "completion_tokens": 25,
    "total_tokens": 125
  }
}
```

---

## 生成嵌入

为输入文本生成向量嵌入。

```bash
curl -X POST http://localhost:8002/osm/api/llm/v1/embeddings \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '{
    "input": ["security analysis", "vulnerability assessment"],
    "model": "text-embedding-3-small"
  }'
```

**请求体：**

| 字段 | 类型 | 必填 | 描述 |
|-------|------|----------|-------------|
| `input` | array | 是 | 要嵌入的字符串数组 |
| `model` | string | 否 | 嵌入模型（默认为供应商的模型） |

**响应：**
```json
{
  "model": "text-embedding-3-small",
  "embeddings": [
    [0.0023, -0.0045, 0.0178, ...],
    [0.0112, -0.0067, 0.0234, ...]
  ],
  "usage": {
    "prompt_tokens": 10,
    "total_tokens": 10
  }
}
```

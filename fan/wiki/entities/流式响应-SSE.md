---
title: 流式响应 SSE
type: entity
entity_type: concept
tags: [SSE, Server-Sent Events, 流式传输, token]
---

# 流式响应 SSE

基于 Server-Sent Events (SSE) 协议的 token 逐步推送模式。客户端请求时设置 `"stream": true`，服务器每生成一个 token 就推送一个 SSE 事件。

## 非流式 vs 流式

| 维度 | 非流式 | 流式 |
|------|--------|------|
| 用户体验 | 等待数秒后一次性显示 | 立刻开始逐字显示 |
| 数据格式 | 单个 JSON 响应 | 多个 `data: {...}\n\n` 事件 |
| 结束标记 | HTTP body 结束 | `data: [DONE]` |
| 缓存条件 | 正常缓存 | 必须 StreamingComplete 才缓存 |

## SSE 数据格式

```
data: {"choices":[{"delta":{"content":"特"}}]}
data: {"choices":[{"delta":{"content":"斯"}}]}
...
data: [DONE]
```

## 在 ExtProc 中的处理

1. **请求头阶段**：检测 `Accept: text/event-stream` → ExpectStreamingResponse
2. **请求体阶段**：检测 JSON 中 `"stream": true` → 确认流式
3. **响应头阶段**：检测 `Content-Type: text/event-stream` → IsStreamingResponse，设置 ModeOverride=STREAMED
4. **响应体阶段**：累积 chunks，解析 delta.content，检测 [DONE] 标记

## 安全机制

流式中途断开（用户关闭浏览器/网络超时）时标记 StreamingAborted=true，防止缓存不完整响应。

## 相关页面

- [[handleResponseBody]] | [[handleResponseHeaders]]
- [[语义缓存]]

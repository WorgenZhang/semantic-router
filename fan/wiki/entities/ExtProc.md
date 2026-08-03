---
title: ExtProc
type: entity
entity_type: concept
tags: [Envoy, gRPC, External Processor, 核心组件]
---

# ExtProc

Envoy gRPC External Processor 服务，实现 `ExternalProcessorServer` 接口。是 vLLM 语义路由系统的核心处理组件。

## 定义

ExtProc 采用经典的 `for { stream.Recv() → switch → handle → sendResponse }` 事件循环模式，在 Envoy 代理的请求/响应链路中插入自定义处理逻辑。

## 架构

```
Envoy gRPC Stream → stream.Recv() → switch req.Request.(type)
  ├─ RequestHeaders  → handleRequestHeaders()  → CONTINUE
  ├─ RequestBody     → handleRequestBody()     → HeaderMutation + BodyMutation
  ├─ ResponseHeaders → handleResponseHeaders() → VSR Headers
  └─ ResponseBody    → handleResponseBody()    → 指标/缓存/幻觉检测
```

## 核心文件

| 文件 | 职责 |
|------|------|
| `server.go` | gRPC Server, RouterService |
| `processor_core.go` | Process() 主循环 |
| `processor_req_header.go` | 请求头处理 |
| `processor_req_body.go` | 请求体处理（核心路由） |
| `processor_res_header.go` | 响应头处理 |
| `processor_res_body.go` | 响应体处理 |

## 错误处理策略

分为四层优先级：
1. 流式中断检测 — StreamingAborted
2. EOF 正常结束 — 返回 nil
3. gRPC 状态码 — Canceled 静默，DeadlineExceeded 记录超时
4. Go context 取消 — 兜底处理

## 相关页面

- [[handleRequestHeaders]] | [[handleRequestBody]] | [[handleResponseHeaders]] | [[handleResponseBody]]
- [[OpenAIRouter]] | [[RequestContext]]
- [[Envoy]]

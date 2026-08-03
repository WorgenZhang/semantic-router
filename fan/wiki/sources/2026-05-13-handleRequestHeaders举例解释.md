---
title: handleRequestHeaders 每步举例解释
date: 2026-05-13
type: source
source_path: raw/notes/2026-05-13-handleRequestHeaders举例解释.md
tags: [handleRequestHeaders, 请求头, trace context, Span]
sources: []
images: 0
image_paths: []
---

# handleRequestHeaders 每步举例解释

## 核心观点

- header 阶段对常规请求只做"观察和记录"，不修改任何东西
- 遍历两次 headers：第一次构建 headerMap（给 TraceContext 用），第二次存入 ctx 并提取业务字段
- 返回 CONTINUE 告诉 Envoy 继续发 body，无 HeaderMutation

## 关键数据流示例

**输入**（Envoy gRPC 发来的 protobuf）：
```
Headers: [:method=POST, :path=/v1/chat/completions, accept=text/event-stream, x-request-id=req-abc-123, traceparent=00-4bf92f...]
```

**输出**（返回给 Envoy）：
```
ProcessingResponse { request_headers { response { status: CONTINUE } } }
```

**内部副作用**：
- ctx.StartTime 记录
- ctx.Headers 全部存储
- ctx.RequestID = "req-abc-123"
- ctx.TraceContext 含 TraceID
- ctx.ExpectStreamingResponse = true
- Span "request_received" 已创建

## 关键概念

- [[handleRequestHeaders]]：第一阶段处理器
- [[分布式追踪]]：ExtractTraceContext + StartSpan
- [[RequestContext]]：存储所有中间状态

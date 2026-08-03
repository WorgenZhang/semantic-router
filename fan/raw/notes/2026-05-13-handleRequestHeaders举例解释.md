@src/semantic-router/pkg/extproc/processor_req_header.go 我想详细知道handleRequestHeaders每一步都在干什么，请每一步给我举具体的例子，例如数据的输入输出，帮助我更好的理解

## `handleRequestHeaders` 逐步详解（带具体数据示例）

假设一个用户发了这样的 HTTP 请求：

```http
POST /v1/chat/completions HTTP/1.1
Host: my-router.example.com:8801
Content-Type: application/json
Accept: text/event-stream
x-request-id: req-abc-123
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Authorization: Bearer sk-xxx

{"model": "auto", "messages": [{"role": "user", "content": "写一个快速排序"}], "stream": true}
```

Envoy 收到后，通过 gRPC 把 **headers 部分** 发给 ExtProc。下面逐步拆解：

---

### Step 1：初始化时间戳（第 157 行）

```go
ctx.StartTime = time.Now()
```

**输入**：无
**输出**：`ctx.StartTime = 2026-04-28T10:30:00.000Z`

**作用**：记录请求到达时间，后续计算总延迟用。

---

### Step 2：第一次遍历 headers — 构建 headerMap（第 160-167 行）

```go
headerMap := make(map[string]string)
for _, h := range v.RequestHeaders.Headers.Headers {
    headerValue := h.Value
    if headerValue == "" && len(h.RawValue) > 0 {
        headerValue = string(h.RawValue)
    }
    headerMap[h.Key] = headerValue
}
```

**输入**（Envoy gRPC 发来的 protobuf）：

```
Headers: [
  {Key: ":method",       Value: "POST"},
  {Key: ":path",         Value: "/v1/chat/completions"},
  {Key: ":authority",    Value: "my-router.example.com:8801"},
  {Key: "content-type",  Value: "application/json"},
  {Key: "accept",        Value: "text/event-stream"},
  {Key: "x-request-id",  Value: "req-abc-123"},
  {Key: "traceparent",   Value: "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"},
  {Key: "authorization", Value: "Bearer sk-xxx"},
]
```

**输出** `headerMap`：

```go
map[string]string{
    ":method":       "POST",
    ":path":         "/v1/chat/completions",
    ":authority":    "my-router.example.com:8801",
    "content-type":  "application/json",
    "accept":        "text/event-stream",
    "x-request-id":  "req-abc-123",
    "traceparent":   "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
    "authorization": "Bearer sk-xxx",
}
```

**为什么单独构建这个 map？** 纯粹为了下一步 `ExtractTraceContext` 能接收 `map[string]string` 参数。

---

### Step 3：提取分布式追踪上下文（第 170 行）

```go
ctx.TraceContext = tracing.ExtractTraceContext(baseCtx, headerMap)
```

**输入**：`headerMap` 中的 `traceparent: "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"`

**输出**：`ctx.TraceContext` 是一个包含以下信息的 Go `context.Context`：

```
TraceID:  4bf92f3577b34da6a3ce929d0e0e4736
SpanID:   00f067aa0ba902b7 (父 span)
Sampled:  true (01)
```

**如果请求没带 `traceparent`**：会生成一个全新的 TraceID。

---

### Step 4：创建根 Span（第 173-175 行）

```go
spanCtx, span := tracing.StartSpan(ctx.TraceContext, tracing.SpanRequestReceived,
    trace.WithSpanKind(trace.SpanKindServer))
ctx.TraceContext = spanCtx
defer span.End()
```

**输入**：上一步的 TraceContext
**输出**：创建了一个新的 Span：

```
Span: {
    Name:     "request_received",
    TraceID:  4bf92f3577b34da6a3ce929d0e0e4736,  // 同一个 Trace
    SpanID:   <新生成, 如 "1234567890abcdef">,
    ParentID: 00f067aa0ba902b7,                   // 来自 traceparent
    Kind:     SERVER,
    Start:    2026-04-28T10:30:00.001Z,
}
```

---

### Step 5：第二次遍历 headers — 存入 ctx 并提取业务字段（第 181-201 行）

```go
for _, h := range requestHeaders.Headers {
    headerValue := h.Value
    if headerValue == "" && len(h.RawValue) > 0 {
        headerValue = string(h.RawValue)
    }
    ctx.Headers[h.Key] = headerValue

    if strings.ToLower(h.Key) == headers.RequestID {  // "x-request-id"
        ctx.RequestID = headerValue
    }
    if h.Key == headers.VSRLooperRequest && headerValue == "true" {
        ctx.LooperRequest = true
    }
}
```

**输入**：同 Step 2 的 header 列表
**输出**：

```go
ctx.Headers = map[string]string{
    ":method":       "POST",
    ":path":         "/v1/chat/completions",
    "accept":        "text/event-stream",
    "x-request-id":  "req-abc-123",
    "traceparent":   "00-...",
    "authorization": "Bearer sk-xxx",
    // ... 全部 headers
}
ctx.RequestID = "req-abc-123"
ctx.LooperRequest = false  // 没有 x-vsr-looper-request header
```

**为什么遍历两次而不合并？** 因为 Span 必须在业务逻辑之前创建（Step 4 用了 `defer span.End()`），而创建 Span 需要 TraceContext（Step 3），TraceContext 需要完整 headerMap（Step 2）。

---

### Step 6：给 Span 打标签（第 204-213 行）

```go
tracing.SetSpanAttributes(span,
    attribute.String(tracing.AttrRequestID, ctx.RequestID))

method := ctx.Headers[":method"]  // "POST"
path := ctx.Headers[":path"]      // "/v1/chat/completions"
tracing.SetSpanAttributes(span,
    attribute.String(tracing.AttrHTTPMethod, method),
    attribute.String(tracing.AttrHTTPPath, path))
```

**输入**：`ctx.RequestID="req-abc-123"`, `method="POST"`, `path="/v1/chat/completions"`
**输出**：Span 的 attributes 被更新：

```
Span attributes: {
    "request.id":  "req-abc-123",
    "http.method": "POST",
    "http.path":   "/v1/chat/completions",
}
```

这些标签在 Jaeger/Grafana 里可以被搜索和筛选。

---

### Step 7：Router Replay API 短路检查（第 215-217 行）

```go
if replayResp := r.handleRouterReplayAPI(method, path); replayResp != nil {
    return replayResp, nil
}
```

**输入**：`method="POST"`, `path="/v1/chat/completions"`
**输出**：`nil`（不匹配 Replay API 路径，继续往下走）

**什么时候会短路？** 如果请求是 `GET /v1/router/replays`，这里直接返回 replay 列表，不进入 body 阶段。

---

### Step 8：检测流式请求（第 220-225 行）

```go
if accept, ok := ctx.Headers["accept"]; ok {
    if strings.Contains(strings.ToLower(accept), "text/event-stream") {
        ctx.ExpectStreamingResponse = true
    }
}
```

**输入**：`ctx.Headers["accept"] = "text/event-stream"`
**输出**：`ctx.ExpectStreamingResponse = true`

**作用**：后续 body 阶段会据此决定是否用 SSE 流式模式处理响应。

---

### Step 9：GET /v1/models 短路检查（第 228-230 行）

```go
if method == "GET" && strings.HasPrefix(path, "/v1/models") {
    return r.handleModelsRequest(path)
}
```

**输入**：`method="POST"`, `path="/v1/chat/completions"`
**输出**：条件不满足，跳过

**什么时候会短路？** 如果请求是 `GET /v1/models`，直接返回可用模型列表（从配置读取），不需要 body。

---

### Step 10：Response API 检查（第 233-267 行）

```go
if r.ResponseAPIFilter != nil && r.ResponseAPIFilter.IsEnabled() && strings.HasPrefix(path, "/v1/responses") {
    // GET/DELETE 直接返回
    // POST 标记 ctx.ResponseAPICtx
}
```

**输入**：`path="/v1/chat/completions"`
**输出**：条件不满足（路径不是 `/v1/responses`），跳过

**什么时候触发？** 比如 `GET /v1/responses/resp_abc123` 会直接返回该 response 对象。

---

### Step 11：构建最终 CONTINUE 响应（第 270-279 行）

```go
response := &ext_proc.ProcessingResponse{
    Response: &ext_proc.ProcessingResponse_RequestHeaders{
        RequestHeaders: &ext_proc.HeadersResponse{
            Response: &ext_proc.CommonResponse{
                Status: ext_proc.CommonResponse_CONTINUE,
            },
        },
    },
}
return response, nil
```

**输入**：无（到这里说明是常规请求）
**输出**（gRPC 返回给 Envoy 的 protobuf）：

```protobuf
ProcessingResponse {
    request_headers {
        response {
            status: CONTINUE   // 告诉 Envoy：headers 我看完了，请继续发 body 给我
            // 注意：没有 header_mutation！headers 阶段不做任何修改
        }
    }
}
```

**Envoy 收到 CONTINUE 后**：开始等待完整 request body（因为配置了 `request_body_mode: BUFFERED`），接收完后发送 `ProcessingRequest_RequestBody` 给 ExtProc。

---

### 完整数据流总结

```
Envoy 发给 ExtProc 的:                    ExtProc 返回给 Envoy 的:
┌─────────────────────────────────┐      ┌─────────────────────────────┐
│ ProcessingRequest {             │      │ ProcessingResponse {        │
│   request_headers {             │      │   request_headers {         │
│     headers: [                  │      │     response {              │
│       ":method" = "POST"        │      │       status: CONTINUE     │
│       ":path" = "/v1/chat/..."  │  ──> │       // 无 mutation        │
│       "accept" = "text/event.." │      │     }                      │
│       "x-request-id" = "req-.." │      │   }                        │
│       "traceparent" = "00-..."  │      │ }                          │
│     ]                           │      └─────────────────────────────┘
│   }                             │
│ }                               │      内部副作用:
└─────────────────────────────────┘      • ctx.StartTime = 记录
                                         • ctx.Headers = 全部存储
                                         • ctx.RequestID = "req-abc-123"
                                         • ctx.TraceContext = 含 TraceID
                                         • ctx.ExpectStreamingResponse = true
                                         • Span "request_received" 已创建
```

**关键理解**：header 阶段对常规请求只做"观察和记录"，不修改任何东西。真正的路由决策（选模型、选 endpoint、改 body）全部延迟到 **body 阶段**处理，因为需要解析 JSON body 才能做语义分类。

我想详细知道handleResponseHeaders每一步都在干什么，请每一步给我举具体的例子，例如数据的输入输出，帮助我更好的理解

## `handleResponseHeaders` 逐步详解

这个函数在 Envoy ExtProc 收到**上游（后端 LLM）返回的 HTTP 响应头**时触发。下面逐步拆解：

---

### 第1步：Looper 内部请求快速返回（L24-43）

```go
if ctx.LooperRequest {
    statusCode = getStatusFromHeaders(v.ResponseHeaders.Headers)
    r.updateRouterReplayStatus(ctx, statusCode, false)
    return CONTINUE
}
```

**场景**：Router 内部发起的"重放"请求（如 A/B 测试或回放流量）。

**输入示例**：

```
ctx.LooperRequest = true
响应头: [":status" = "200"]
```

**输出**：直接返回 `CONTINUE`，不做任何额外处理。相当于"这是内部流量，不需要加额外 header，直接放行"。

---

### 第2步：提取 HTTP 状态码 + 判断流式响应（L44-63）

```go
ctx.IsStreamingResponse = isStreamingContentType(headers)
statusCode = getStatusFromHeaders(headers)
isSuccessful = statusCode >= 200 && statusCode < 300
```

**输入示例**：

```
响应头:
  ":status" = "200"
  "content-type" = "text/event-stream"   ← SSE 流式
```

**输出**：

```
statusCode = 200
isSuccessful = true
ctx.IsStreamingResponse = true
```

**另一个示例（失败）**：

```
响应头:
  ":status" = "503"
  "content-type" = "application/json"
```

**输出**：

```
statusCode = 503
isSuccessful = false
ctx.IsStreamingResponse = false
→ 调用 metrics.RecordRequestError("llama-3-70b", "upstream_5xx")
```

---

### 第3步：结束上游请求的 tracing span（L66-78）

```go
tracing.SetSpanAttributes(ctx.UpstreamSpan, attribute.Int("http.status_code", 200))
ctx.UpstreamSpan.End()
```

**场景**：在请求头阶段（`createRoutingResponse`）创建了一个 OpenTelemetry span 来追踪"请求发到上游的耗时"。现在响应头回来了，span 结束。

**输入**：`ctx.UpstreamSpan` 是一个活跃的 span，`statusCode = 503`

**输出**：

```
span 属性: http.status_code = 503
span 状态: Error("upstream request failed")
span 被 End()，记录到 Jaeger/Tempo 等追踪系统
ctx.UpstreamSpan = nil  ← 防止重复 End
```

---

### 第4步：TTFT（Time To First Token）记录（L83-92）

```go
if !ctx.IsStreamingResponse && !ctx.TTFTRecorded {
    ttft = time.Since(ctx.ProcessingStartTime).Seconds()
    metrics.RecordModelTTFT(ctx.RequestModel, ttft)
    latency.UpdateTTFT(ctx.RequestModel, ttft)
}
```

**场景**：对于**非流式响应**，收到响应头 ≈ 收到第一个 token（因为整个 body 一次性返回）。

**输入示例**：

```
ctx.IsStreamingResponse = false
ctx.ProcessingStartTime = 10:00:00.000
当前时间 = 10:00:00.350
ctx.RequestModel = "llama-3-8b"
```

**输出**：

```
ttft = 0.350 秒
→ Prometheus 直方图记录: model="llama-3-8b", ttft=0.35
→ 延迟缓存更新: latency.UpdateTTFT("llama-3-8b", 0.35)
   （供 latency_aware 策略选模型时参考 P50/P99）
ctx.TTFTRecorded = true  ← 防止重复计算
```

**注意**：流式响应的 TTFT 在 `handleResponseBody` 中第一个 SSE chunk 到达时才记录。

---

### 第5步：更新 Router Replay 状态（L95）

```go
r.updateRouterReplayStatus(ctx, statusCode, ctx.IsStreamingResponse)
```

**作用**：将此次请求的结果（成功/失败、是否流式）记录到 replay 系统，用于后续分析和流量回放。

---

### 第6步：构建 VSR 决策追踪响应头（L98-309）

这是最长的一段，只在 `isSuccessful && !ctx.VSRCacheHit` 时执行。目的是**把路由决策的元数据通过响应头返回给客户端**，便于可观测性和调试。

**输入示例**：

```
ctx.VSRSelectedCategory = "code_generation"
ctx.VSRSelectedDecisionName = "route-to-large-model"
ctx.VSRSelectedDecisionConfidence = 0.9234
ctx.VSRSelectedModel = "llama-3-70b"
ctx.VSRInjectedSystemPrompt = true
ctx.VSRMatchedKeywords = ["implement", "function"]
ctx.VSRMatchedComplexity = ["high"]
ctx.VSRContextTokenCount = 4096
ctx.RouterReplayID = "replay-abc-123"
```

**输出（附加到响应头）**：

```http
x-vsr-selected-category: code_generation
x-vsr-selected-decision: route-to-large-model
x-vsr-selected-confidence: 0.9234
x-vsr-selected-model: llama-3-70b
x-vsr-injected-system-prompt: true
x-vsr-matched-keywords: implement,function
x-vsr-matched-complexity: high
x-vsr-context-token-count: 4096
x-vsr-router-replay-id: replay-abc-123
```

客户端收到这些 header 后可以：

- 前端 dashboard 展示路由决策
- 调试为什么选了某个模型
- 追踪信号分类结果（关键词匹配、领域分类等）

---

### 第7步：流式响应的 Mode Override（L326-330）

```go
if ctx.IsStreamingResponse {
    response.ModeOverride = &ProcessingMode{
        ResponseBodyMode: ProcessingMode_STREAMED,
    }
}
```

**场景**：当响应是 SSE 流式时，告诉 Envoy "把后续的 response body 也逐 chunk 转发给 ExtProc"。

**输入**：`ctx.IsStreamingResponse = true`

**输出**：Envoy 配置被覆盖为 `STREAMED` 模式。后续每个 SSE chunk（`data: {"choices":[...]}\n\n`）都会触发 `handleResponseBody`，从而可以在第一个 chunk 到达时测量真正的 TTFT。

如果不设置这个，Envoy 会缓冲整个响应体再转发，流式就失去意义了。

---

### 整体数据流图

```
Client ← Envoy ← LLM (upstream)
              ↕
         ExtProc (这个函数)

LLM 返回:
  HTTP/2 200
  content-type: text/event-stream

本函数处理后返回给 Envoy:
  action: CONTINUE (放行响应)
  附加 header: x-vsr-selected-model: llama-3-70b, ...
  mode_override: STREAMED (让后续 body chunk 也发给 ExtProc)
```

---

### 辅助函数

| 函数                       | 作用                          | 示例                                               |
| -------------------------- | ----------------------------- | -------------------------------------------------- |
| `getStatusFromHeaders`   | 从 `:status` 伪头提取状态码 | `":status"="429"` → `429`                     |
| `getModelFromCtx`        | 安全获取模型名                | `ctx.RequestModel=""` → `"unknown"`           |
| `isStreamingContentType` | 检测 content-type 是否为 SSE  | `"text/event-stream; charset=utf-8"` → `true` |

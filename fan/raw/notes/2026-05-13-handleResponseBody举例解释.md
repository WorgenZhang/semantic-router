@src/semantic-router/pkg/extproc/processor_res_body.go 我想详细知道handleResponseBody每一步都在干什么，请每一步给我举具体的例子，例如数据的输入输出，帮助我更好的理解

---

# handleResponseBody 每步举例详解

## 总览

`handleResponseBody` 是 ExtProc 四阶段的最后一步。当 LLM 后端（如 vLLM）返回响应 body 后，Envoy 将响应体交给这个函数处理。它负责：指标记录、缓存更新、格式转换、幻觉检测、记忆提取等。

---

## 函数输入

```
输入参数:
- v *ext_proc.ProcessingRequest_ResponseBody  → 包含 LLM 返回的原始 response body 字节
- ctx *RequestContext                         → 请求上下文（贯穿整个请求生命周期的状态）

例如 v.ResponseBody.Body 的内容:
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "gpt-4",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "Paris is the capital of France."},
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 15, "completion_tokens": 8, "total_tokens": 23}
}
```

---

## 第1步：计算完成延迟 & 递减活跃请求数 (第23-27行)

```go
completionLatency := time.Since(ctx.StartTime)
defer metrics.DecrementModelActiveRequests(ctx.RequestModel)
```

**做什么：** 计算从请求发出到响应返回的总耗时，并在函数返回时递减该模型的活跃请求计数。

**举例：**
```
输入: ctx.StartTime = 10:00:00.000, 当前时间 = 10:00:01.250
      ctx.RequestModel = "llama-3-70b"

计算: completionLatency = 1.25s
效果: 函数返回时 llama-3-70b 的 active_requests 从 5 降为 4
      （用于队列深度估计和负载均衡决策）
```

---

## 第2步：Looper 内部请求短路返回 (第30-47行)

```go
if ctx.LooperRequest {
    responseBody := v.ResponseBody.Body
    r.attachRouterReplayResponse(ctx, responseBody, true)
    return CONTINUE
}
```

**做什么：** 如果这是 Looper（循环路由器）发起的内部请求，直接捕获响应体并继续，不做后续处理。Looper 会自己处理这个响应。

**举例：**
```
场景: Looper 决策需要先调用一个分类模型，再根据分类结果路由到不同的 LLM。
      这个分类模型的响应就是 LooperRequest。

输入: ctx.LooperRequest = true
      responseBody = {"choices":[{"message":{"content":"category: technical"}}]}

输出: 直接返回 CONTINUE，不做缓存/指标/幻觉检测等后续步骤
      响应体被保存到 ctx.RouterReplayRecorder 用于审计追踪
```

---

## 第3步：Anthropic 格式转换 (第52-65行)

```go
if ctx.APIFormat == config.APIFormatAnthropic {
    transformedBody, err := anthropic.ToOpenAIResponseBody(responseBody, ctx.RequestModel)
    responseBody = transformedBody
    anthropicTransformed = true
}
```

**做什么：** 如果请求被路由到了 Anthropic 后端（如 Claude），需要把 Anthropic 格式的响应转换回 OpenAI 格式，因为客户端发送的是 OpenAI 格式的请求。

**举例：**
```
输入 (Anthropic 原始响应):
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "content": [{"type": "text", "text": "Paris is the capital of France."}],
  "model": "claude-sonnet-4-20250514",
  "stop_reason": "end_turn",
  "usage": {"input_tokens": 15, "output_tokens": 8}
}

输出 (转换为 OpenAI 格式):
{
  "id": "chatcmpl-msg_01XFDUDYJgAACzvnptvVoYEL",
  "object": "chat.completion",
  "model": "claude-sonnet-4-20250514",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "Paris is the capital of France."},
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 15, "completion_tokens": 8, "total_tokens": 23}
}

为什么需要: 客户端只认识 OpenAI 格式，不知道后端是 Anthropic
```

---

## 第4步：流式响应处理分支 (第69-128行)

如果 `ctx.IsStreamingResponse == true`，进入流式处理逻辑：

### 4a. 记录 TTFT (首 token 时间) (第70-79行)

```go
if !ctx.TTFTRecorded && !ctx.ProcessingStartTime.IsZero() {
    ttft := time.Since(ctx.ProcessingStartTime).Seconds()
    metrics.RecordModelTTFT(ctx.RequestModel, ttft)
    latency.UpdateTTFT(ctx.RequestModel, ttft)
}
```

**举例：**
```
场景: 流式响应的第一个 chunk 到达

输入: ctx.ProcessingStartTime = 10:00:00.000 (请求发送时间)
      当前时间 = 10:00:00.350
      ctx.RequestModel = "llama-3-70b"
      ctx.TTFTRecorded = false

计算: ttft = 0.35s (350ms，即第一个 token 返回的延迟)

效果: - Prometheus 指标记录 TTFT = 0.35s
      - latency 模块更新百分位数缓存（用于 latency_aware 路由策略）
      - ctx.TTFTRecorded = true (后续 chunk 不再记录)
```

### 4b. 累积流式 chunks (第82-91行)

```go
ctx.StreamingChunks = append(ctx.StreamingChunks, chunk)
r.parseStreamingChunk(chunk, ctx)
```

**举例：**
```
第1个 chunk:
  data: {"id":"chatcmpl-abc","choices":[{"delta":{"role":"assistant","content":"Paris"}}]}

第2个 chunk:
  data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":" is the"}}]}

第3个 chunk:
  data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":" capital of France."}}]}

第4个 chunk:
  data: [DONE]

累积效果:
  ctx.StreamingChunks = ["data: {chunk1}\n\n", "data: {chunk2}\n\n", ...]
  ctx.StreamingContent = "Paris is the capital of France."  (逐步拼接)
  ctx.StreamingMetadata = {"id": "chatcmpl-abc", "model": "llama-3-70b", "created": 1700000000}
```

### 4c. 流完成后缓存重建 (第94-115行)

```go
if strings.Contains(chunk, "data: [DONE]") {
    ctx.StreamingComplete = true
    metrics.RecordModelCompletionLatency(ctx.RequestModel, completionLatency)
    r.cacheStreamingResponse(ctx)  // 重建完整响应并缓存
}
```

**举例：**
```
当收到 [DONE] 标记时:
  - 将 ctx.StreamingContent = "Paris is the capital of France." 
  - 加上 ctx.StreamingMetadata (id, model, created, finish_reason)
  - 重建为标准 ChatCompletion JSON:
    {
      "id": "chatcmpl-abc",
      "object": "chat.completion",
      "model": "llama-3-70b",
      "choices": [{"index": 0, "message": {"role": "assistant", "content": "Paris is the capital of France."}, "finish_reason": "stop"}],
      "usage": {"prompt_tokens": 15, "completion_tokens": 8, "total_tokens": 23}
    }
  - 存入语义缓存（下次相同/相似请求可直接命中）
```

### 4d. 流式每个 chunk 都返回 CONTINUE (第118-128行)

```
每个流式 chunk 都立即转发给客户端，不阻塞。
只有收到 [DONE] 时才做缓存等收尾工作。
```

**注意：如果是流式响应，到这里就 return 了，不会执行后面的非流式逻辑。**

---

## 第5步：解析 Token 用量 (第131-135行) —— 仅非流式

```go
var parsed openai.ChatCompletion
json.Unmarshal(responseBody, &parsed)
promptTokens := int(parsed.Usage.PromptTokens)
completionTokens := int(parsed.Usage.CompletionTokens)
```

**举例：**
```
输入 responseBody:
{
  "usage": {"prompt_tokens": 150, "completion_tokens": 45, "total_tokens": 195}
}

输出:
  promptTokens = 150
  completionTokens = 45
```

---

## 第6步：向 Rate Limiter 上报 Token 用量 (第139-146行)

```go
r.RateLimiter.Report(*ctx.RateLimitCtx, ratelimit.TokenUsage{
    InputTokens:  promptTokens,
    OutputTokens: completionTokens,
    TotalTokens:  promptTokens + completionTokens,
})
```

**做什么：** 将本次请求消耗的 token 数上报给限流器，用于 TPM (tokens per minute) 预算追踪。

**举例：**
```
输入: ctx.RateLimitCtx = {BucketKey: "user-123/gpt-4", TPMLimit: 100000}
      promptTokens = 150, completionTokens = 45

效果: user-123 对 gpt-4 的 TPM 预算从剩余 99800 降为 99605
      如果预算耗尽，后续请求会被限流（返回 429 Too Many Requests）
```

---

## 第7步：记录详细 Token 指标 + TPOT + 窗口化指标 (第149-177行)

```go
metrics.RecordModelTokensDetailed(ctx.RequestModel, promptTokens, completionTokens)
metrics.RecordModelCompletionLatency(ctx.RequestModel, completionLatency.Seconds())

if completionTokens > 0 {
    timePerToken := completionLatency.Seconds() / float64(completionTokens)
    metrics.RecordModelTPOT(ctx.RequestModel, timePerToken)
    latency.UpdateTPOT(ctx.RequestModel, timePerToken)
}

metrics.RecordModelWindowedRequest(ctx.RequestModel, completionLatency, promptTokens, completionTokens, false, false)
```

**举例：**
```
输入: ctx.RequestModel = "llama-3-70b"
      completionLatency = 1.25s
      promptTokens = 150, completionTokens = 45

计算:
  TPOT (Time Per Output Token) = 1.25 / 45 = 0.0278s/token (27.8ms/token)

效果:
  1. Prometheus 记录: model_tokens_prompt{model="llama-3-70b"} += 150
                      model_tokens_completion{model="llama-3-70b"} += 45
  2. Prometheus 记录: model_completion_latency{model="llama-3-70b"} = 1.25s
  3. Prometheus 记录: model_tpot{model="llama-3-70b"} = 0.0278
  4. latency 模块更新 TPOT 百分位数缓存 → 用于 latency_aware 路由策略选择更快的模型
  5. 窗口化指标用于实时负载均衡决策（滑动窗口内的 QPS、延迟、吞吐）
```

---

## 第8步：计算并记录成本 (第180-211行)

```go
promptRatePer1M, completionRatePer1M, currency, ok := r.Config.GetModelPricing(ctx.RequestModel)
costAmount := (float64(promptTokens)*promptRatePer1M + float64(completionTokens)*completionRatePer1M) / 1_000_000.0
metrics.RecordModelCost(ctx.RequestModel, currency, costAmount)
```

**做什么：** 根据配置的模型定价，计算本次请求的实际费用。

**举例：**
```
输入: ctx.RequestModel = "gpt-4"
      定价配置: prompt_rate = $30/1M tokens, completion_rate = $60/1M tokens
      promptTokens = 150, completionTokens = 45

计算:
  cost = (150 * 30 + 45 * 60) / 1,000,000
       = (4500 + 2700) / 1,000,000
       = $0.0072

输出日志 (LogEvent "llm_usage"):
{
  "request_id": "req-abc-123",
  "model": "gpt-4",
  "prompt_tokens": 150,
  "completion_tokens": 45,
  "total_tokens": 195,
  "completion_latency_ms": 1250,
  "cost": 0.0072,
  "currency": "USD"
}
```

---

## 第9步：更新语义缓存 (第214-228行)

```go
ttlSeconds := r.Config.GetCacheTTLSecondsForDecision(ctx.VSRSelectedDecisionName)
err := r.Cache.UpdateWithResponse(ctx.RequestID, responseBody, ttlSeconds)
```

**做什么：** 将完整的响应存入语义缓存。在 handleRequestBody 阶段已经用 `AddPendingRequest` 注册了请求（包含 embedding），现在用实际响应来完成这个缓存条目。

**举例：**
```
输入: ctx.RequestID = "req-abc-123"
      ctx.VSRSelectedDecisionName = "coding_assistant"
      配置: coding_assistant 决策的 cache_ttl = 3600 (1小时)
      responseBody = {"choices":[{"message":{"content":"Paris is the capital..."}}]}

效果: 语义缓存中 req-abc-123 从 "pending" 状态变为 "completed"
      存储: embedding → response 的映射
      TTL = 3600s

后续: 如果有新请求 "What is the capital of France?" 
      其 embedding 与缓存条目相似度 > 阈值 → 直接返回缓存响应，不调 LLM
```

---

## 第10步：异步记忆提取 (第231-278行)

```go
autoStoreEnabled := extractAutoStore(ctx)
if r.MemoryExtractor != nil && autoStoreEnabled {
    go func() {
        r.MemoryExtractor.ProcessResponse(bgCtx, sessionID, userID, history)
    }()
}
```

**做什么：** 如果启用了自动记忆存储（Response API 特性），异步提取对话中的关键信息存入长期记忆。不阻塞响应返回。

**举例：**
```
输入: 
  autoStoreEnabled = true (决策配置或全局配置启用)
  sessionID = "conv-xyz-789"
  userID = "user-123"
  history = [
    {role: "user", content: "My name is Alice and I work at Google on TPU compiler."},
    {role: "assistant", content: "Nice to meet you, Alice! That sounds like fascinating work..."},
    {role: "user", content: "What framework should I use for model serving?"},  ← 当前 turn
    {role: "assistant", content: "Given your TPU background, I'd recommend..."}  ← 当前响应
  ]

效果 (异步执行，不阻塞):
  MemoryExtractor 用 LLM 分析对话，提取记忆:
  - fact: "User's name is Alice"
  - fact: "User works at Google on TPU compiler team"
  - preference: "User is interested in model serving frameworks"
  
  这些记忆存储到向量数据库，下次对话可以检索到:
  "Hi, welcome back Alice! Last time you asked about model serving for TPU..."
```

---

## 第11步：Response API 格式转换 (第280-291行)

```go
if ctx.ResponseAPICtx != nil && ctx.ResponseAPICtx.IsResponseAPIRequest && r.ResponseAPIFilter != nil {
    translatedBody, err := r.ResponseAPIFilter.TranslateResponse(ctx.TraceContext, ctx.ResponseAPICtx, responseBody)
    finalBody = translatedBody
}
```

**做什么：** 如果客户端发送的是 Response API 格式的请求（`/v1/responses`），需要把 OpenAI Chat Completion 格式的响应转换为 Response API 格式。

**举例：**
```
输入 (标准 Chat Completion 格式):
{
  "id": "chatcmpl-abc123",
  "choices": [{"message": {"role": "assistant", "content": "Paris is the capital..."}}]
}

输出 (Response API 格式):
{
  "id": "resp_abc123",
  "object": "response",
  "status": "completed",
  "output": [
    {
      "type": "message",
      "role": "assistant",
      "content": [{"type": "output_text", "text": "Paris is the capital..."}]
    }
  ],
  "usage": {"input_tokens": 150, "output_tokens": 45, "total_tokens": 195}
}
```

---

## 第12步：构建响应 Body Mutation (第293-308行)

```go
if anthropicTransformed || isResponseAPIRequest {
    bodyMutation = &ext_proc.BodyMutation{Mutation: &ext_proc.BodyMutation_Body{Body: finalBody}}
    headerMutation = &ext_proc.HeaderMutation{RemoveHeaders: []string{"content-length"}}
}
```

**做什么：** 如果响应体被修改过（Anthropic 转换或 Response API 转换），告诉 Envoy 用新的 body 替换原始 body，并删除 `content-length` header（因为 body 大小变了，让 Envoy 重新计算）。

**举例：**
```
场景: Anthropic 响应被转换为 OpenAI 格式

原始 body 大小: 450 bytes (Anthropic 格式)
转换后 body 大小: 380 bytes (OpenAI 格式)

效果:
  - bodyMutation.Body = 转换后的 380 bytes JSON
  - headerMutation.RemoveHeaders = ["content-length"]
  - Envoy 会自动设置新的 content-length: 380
```

---

## 第13步：幻觉检测 (第311-322行)

```go
if hallucinationResponse := r.performHallucinationDetection(ctx, responseBody); hallucinationResponse != nil {
    return hallucinationResponse, nil  // 阻断响应
}

if hallucinationConfig.Enabled {
    r.checkUnverifiedFactualResponse(ctx)
}
```

**做什么：** 
1. 调用幻觉检测器（可能用 NLI 模型）检查 LLM 的回答是否与提供的上下文矛盾
2. 检查是否是未经验证的事实性回答

**举例：**
```
场景: 用户问 "根据文档，公司2024年的收入是多少？"
      文档上下文: "2024年公司总收入为5.2亿美元"
      LLM 回答: "根据文档，公司2024年的收入为8.7亿美元。"  ← 幻觉!

检测过程:
  - 提取 assistantContent = "根据文档，公司2024年的收入为8.7亿美元。"
  - 提取 ctx.ToolResultsContext = "2024年公司总收入为5.2亿美元"（RAG 检索结果）
  - NLI 模型判定: CONTRADICTION, confidence=0.95
  
结果:
  ctx.HallucinationDetected = true
  ctx.HallucinationConfidence = 0.95
  ctx.HallucinationSpans = ["8.7亿美元"]  (与上下文矛盾的部分)

如果 action="block":
  直接返回 502 错误响应，不把幻觉内容发给客户端

如果 action="warn" (更常见):
  继续到第14步处理
```

---

## 第14步：应用幻觉警告 (第325-365行)

```go
if ctx.HallucinationDetected {
    modifiedBody, response = r.applyHallucinationWarning(response, ctx, modifiedBody)
}
if ctx.UnverifiedFactualResponse {
    modifiedBody, response = r.applyUnverifiedFactualWarning(response, ctx, modifiedBody)
}
```

**做什么：** 根据配置的 action（header/body/none），在响应中添加幻觉警告信息。

**举例：**
```
Action = "header" 时:
  响应 header 中添加:
    X-Hallucination-Detected: true
    X-Hallucination-Confidence: 0.95
  body 不变

Action = "body" 时:
  原始 body:
  {"choices":[{"message":{"content":"收入为8.7亿美元"}}]}
  
  修改后 body:
  {"choices":[{"message":{"content":"收入为8.7亿美元\n\n⚠️ Warning: This response may contain hallucinated content (confidence: 0.95). Unsupported spans: [\"8.7亿美元\"]"}}]}

Action = "none" 时:
  只记录指标，不修改响应
```

---

## 第15步：更新 Router Replay 幻觉状态 (第368行)

```go
r.updateRouterReplayHallucinationStatus(ctx)
```

**做什么：** 把幻觉检测结果写入 Router Replay 记录，用于审计和调试。

**举例：**
```
Router Replay 记录更新:
{
  "replay_id": "replay-abc-123",
  "request": {...},
  "response": {...},
  "hallucination": {
    "detected": true,
    "confidence": 0.95,
    "unsupported_spans": ["8.7亿美元"],
    "action_taken": "warn_header"
  }
}

用途: 运维人员可以回放和分析路由决策的质量
```

---

## 第16步：附加 Replay 响应并返回 (第371-373行)

```go
r.attachRouterReplayResponse(ctx, finalBody, true)
return response, nil
```

**做什么：** 将最终响应体保存到 Replay 记录器（审计日志），然后返回最终的 ProcessingResponse 给 Envoy。

**举例：**
```
最终返回给 Envoy 的 ProcessingResponse:
{
  Response: ResponseBody {
    CommonResponse {
      Status: CONTINUE,                    // 告诉 Envoy 继续把响应发给客户端
      HeaderMutation: {                    // 可能修改响应 header
        RemoveHeaders: ["content-length"],
        SetHeaders: [
          {Key: "X-Hallucination-Detected", Value: "true"}
        ]
      },
      BodyMutation: {                      // 可能修改响应 body
        Body: <修改后的 JSON bytes>
      }
    }
  }
}

Envoy 收到后:
  1. 如果有 BodyMutation → 用新 body 替换原始响应 body
  2. 如果有 HeaderMutation → 添加/删除响应 header
  3. Status=CONTINUE → 将修改后的响应转发给客户端
```

---

## 完整流程图示

```
LLM 后端响应到达
       │
       ▼
┌─────────────────────┐
│ 1. 计算延迟 + defer │
│    递减活跃请求      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐    是
│ 2. Looper 请求?     │───────→ 捕获响应 → 返回 CONTINUE
└─────────┬───────────┘
          │ 否
          ▼
┌─────────────────────┐    是
│ 3. Anthropic 格式?  │───────→ 转换为 OpenAI 格式
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐    是
│ 4. 流式响应?        │───────→ 记录TTFT → 累积chunks → 
└─────────┬───────────┘         [DONE]时重建+缓存 → 返回 CONTINUE
          │ 否
          ▼
┌─────────────────────────────────┐
│ 5. 解析 Token 用量              │
│ 6. 上报 Rate Limiter            │
│ 7. 记录指标 (TPOT, 窗口化)      │
│ 8. 计算成本                     │
│ 9. 更新语义缓存                 │
│ 10. 异步记忆提取 (go routine)   │
│ 11. Response API 格式转换       │
│ 12. 构建 Body/Header Mutation   │
│ 13. 幻觉检测                    │
│ 14. 应用幻觉警告                │
│ 15. 更新 Replay 幻觉状态        │
│ 16. 附加 Replay 响应 → 返回     │
└─────────────────────────────────┘
```

---

## 关键设计思想

| 设计点 | 说明 |
|--------|------|
| **延迟不阻塞** | 记忆提取用 goroutine 异步执行，不增加响应延迟 |
| **流式 vs 非流式** | 两条完全不同的路径，流式需要累积重建，非流式直接解析 |
| **格式透明** | 客户端永远看到 OpenAI/Response API 格式，不知道后端是 Anthropic/vLLM |
| **安全检测后置** | 幻觉检测在缓存之后，这样即使检测失败也已经缓存了原始响应 |
| **Mutation 机制** | 通过 Envoy ExtProc 的 BodyMutation/HeaderMutation 修改响应，而非重新构建整个 HTTP 响应 |
| **指标全面** | 每个请求记录：延迟、Token数、TPOT、TTFT、成本、幻觉率 —— 驱动路由决策 |

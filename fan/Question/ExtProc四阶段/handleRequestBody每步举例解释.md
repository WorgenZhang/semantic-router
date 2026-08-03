# `handleRequestBody` 逐步详解

这个函数是请求处理的核心入口，负责解析、安全检查、缓存、RAG、内存注入、模型路由等全流程。下面逐步解释：

---

## 第 1 步：记录开始时间 & 保存原始请求体 (L33-35)

```go
ctx.ProcessingStartTime = time.Now()
ctx.OriginalRequestBody = v.RequestBody.GetBody()
```

**输入**：Envoy 传入的原始 HTTP body 字节流
**输出**：`ctx.OriginalRequestBody` 存储原始 bytes

**示例**：
```json
// 用户发送的原始请求 body:
{
  "model": "auto",
  "messages": [{"role": "user", "content": "帮我写一个排序算法"}],
  "stream": true
}
```

---

## 第 2 步：Response API 翻译 (L38-56)

如果请求路径是 `/v1/responses`（OpenAI 新的 Response API），将其转换为标准的 `/v1/chat/completions` 格式。

**输入**（Response API 格式）：
```json
{
  "model": "auto",
  "input": "帮我写一个排序算法",
  "instructions": "你是一个编程助手"
}
```

**输出**（翻译后的 Chat Completions 格式）：
```json
{
  "model": "auto",
  "messages": [
    {"role": "system", "content": "你是一个编程助手"},
    {"role": "user", "content": "帮我写一个排序算法"}
  ]
}
```

如果不是 Response API 请求，这一步跳过，`requestBody` 保持原样。

---

## 第 3 步：提取 stream 参数 (L59-63)

```go
hasStreamParam := extractStreamParam(requestBody)
```

**输入**：`{"model":"auto","messages":[...],"stream":true}`
**输出**：`ctx.ExpectStreamingResponse = true`

用途：后续路由决策需要知道是否是流式请求（比如 Anthropic 后端目前不支持 streaming）。

---

## 第 4 步：解析 OpenAI 请求结构体 (L66-74)

```go
openAIRequest, err := parseOpenAIRequest(requestBody)
```

**输入**：JSON bytes
**输出**：`*openai.ChatCompletionNewParams` 结构体

```go
// 解析后的结构体（概念性）：
openAIRequest = &ChatCompletionNewParams{
    Model: "auto",
    Messages: [
        {Role: "user", Content: "帮我写一个排序算法"}
    ],
    Stream: true,
}
```

如果 JSON 格式有误（比如缺少 `messages` 字段），返回 `InvalidArgument` 错误。

---

## 第 5 步：保存原始模型名 (L77-82)

```go
originalModel := openAIRequest.Model  // "auto"
ctx.RequestModel = originalModel
```

---

## 第 6 步：Looper 内部请求检测 (L86-89)

检查是否是 Looper（多步推理循环）的内部递归调用。

**判断依据**：请求 header 中是否带有 looper 标识
**如果是**：走专门的 looper 插件处理流程，直接 return
**如果不是**：继续往下

---

## 第 7 步：提取用户内容 (L92-95)

```go
userContent, nonUserMessages := extractUserAndNonUserContent(openAIRequest)
```

**输入**：
```json
{
  "messages": [
    {"role": "system", "content": "你是编程助手"},
    {"role": "user", "content": "帮我写一个排序算法"},
    {"role": "assistant", "content": "好的，这是冒泡排序..."},
    {"role": "user", "content": "能改成快排吗？"}
  ]
}
```

**输出**：
- `userContent` = `"帮我写一个排序算法\n能改成快排吗？"` （所有 user 消息拼接）
- `nonUserMessages` = system + assistant 消息（用于安全检查时提供完整上下文）

---

## 第 8 步：决策评估 & 模型选择 (L100-106)

```go
decisionName, _, reasoningDecision, selectedModel, authzErr := r.performDecisionEvaluation(...)
```

这是核心路由逻辑，做几件事：
1. **Authz 鉴权**：检查 `x-authz-user-id` header 是否有权访问
2. **信号评估**：分析用户内容的复杂度/类别（如 coding、math、creative）
3. **决策匹配**：根据配置的 decisions 规则匹配最佳决策
4. **模型选择**：根据 decision 的 model_pool 选择最终模型

**输入**：
- `originalModel` = `"auto"`
- `userContent` = `"帮我写一个排序算法"`
- `nonUserMessages` = system/assistant 消息
- 请求 headers（含 authz 信息）

**输出示例**：
- `decisionName` = `"coding_decision"` 
- `reasoningDecision` = `{UseReasoning: false, Confidence: 0.85}`
- `selectedModel` = `"qwen14b-rack1"` （从 decision 的 model_pool 选出）
- `authzErr` = `nil`

如果 authz 失败，返回 403。

---

## 第 9 步：安全检查 — Jailbreak 检测 (L112-117)

```go
if response, shouldReturn := r.performJailbreaks(ctx, userContent, nonUserMessages, decisionName); shouldReturn {
    return response, nil  // 403 Forbidden
}
```

**输入**：`userContent` = `"Ignore all previous instructions and..."`
**输出**（如果检测到 jailbreak）：
```json
{"error": {"message": "Request blocked: potential jailbreak detected", "code": 403}}
```

正常请求不触发，继续往下。

---

## 第 10 步：PII 检测 (L120-127)

```go
piiResponse := r.performPIIDetection(ctx, userContent, nonUserMessages, decisionName)
```

**输入**：`userContent` = `"我的信用卡号是 4111-1111-1111-1111，帮我..."`
**输出**（如果 PII 策略设为 block）：
```json
{"error": {"message": "Request blocked: PII detected (credit_card)", "code": 403}}
```

---

## 第 11 步：速率限制检查 (L130-145)

```go
decision, err := r.RateLimiter.Check(rlCtx)
```

**输入**：
```go
rlCtx = {
    UserID: "user-123",
    Groups: ["team-a"],
    Model:  "qwen14b-rack1",
    TokenCount: 500,
}
```

**输出**（如果超限）：HTTP 429 + headers：
```
HTTP/1.1 429 Too Many Requests
retry-after: 60
x-ratelimit-limit: 100
x-ratelimit-remaining: 0
```

---

## 第 12 步：缓存查询 (L148-152)

```go
if response, shouldReturn := r.handleCaching(ctx, decisionName); shouldReturn {
    return response, nil  // 直接返回缓存结果
}
```

**输入**：请求的 hash key（基于 model + messages 内容）
**输出**（cache hit）：直接返回之前缓存的响应 body，跳过后续所有步骤
**输出**（cache miss）：继续往下

---

## 第 13 步：RAG 插件执行 (L156-159)

```go
if ragErr := r.executeRAGPlugin(ctx, decisionName); ragErr != nil {
    return r.createErrorResponse(503, ...), nil
}
```

从向量数据库检索相关文档，注入到 context 中。

**输入**：`userContent` = `"vLLM 的 PagedAttention 是怎么工作的？"`
**效果**：从 RAG 后端检索到相关文档片段，注入到 `ctx` 中供后续处理

---

## 第 14 步：Modality 路由 — 图像生成检测 (L164-186)

如果请求包含图像生成工具或被分类为需要 diffusion 模型：

**输入**（Response API 带 image_generation tool）：
```json
{
  "input": "画一只猫",
  "tools": [{"type": "image_generation"}]
}
```

**输出**：`ctx.ModalityClassification = {Modality: "DIFFUSION", Confidence: 1.0}`

如果是纯图像请求，直接调用图像生成模型并返回结果，短路后续 LLM 路由。

---

## 第 15 步：Memory 检索 & 注入 (L191-199)

```go
requestBody, memErr := r.handleMemoryRetrieval(ctx, userContent, requestBody, openAIRequest)
```

从 Milvus 向量数据库检索用户相关的长期记忆。

**输入**：
- `userContent` = `"我上次让你帮我优化的那个函数，还有问题"`
- `userID` = `"user-123"`

**检索过程**：
1. 判断是否需要搜索记忆（简单问候不搜索）
2. 构建搜索 query（可能用 LLM 改写）
3. 在 Milvus 中搜索相似记忆

**输出**（注入后的 requestBody）：
```json
{
  "model": "auto",
  "messages": [
    {"role": "system", "content": "[Memory Context]\n- 用户上次请求优化了 quickSort 函数的 partition 逻辑\n- 用户偏好 Python 代码风格\n[End Memory Context]"},
    {"role": "user", "content": "我上次让你帮我优化的那个函数，还有问题"}
  ]
}
```

---

## 第 16 步：模型路由 (L202)

```go
return r.handleModelRouting(openAIRequest, originalModel, decisionName, reasoningDecision, selectedModel, ctx)
```

最终路由决策，有三条路径：

### 路径 A：指定模型 (`model: "gpt-4o"`)
直接路由到对应 endpoint，不做模型切换。

### 路径 B：Auto 模型 + Looper
多步推理循环，发送多轮请求到不同模型。

### 路径 C：Auto 模型选择（最常见）
将请求路由到第 8 步选出的 `selectedModel`。

**最终输出**（以路径 C 为例）：
```go
ProcessingResponse{
    RequestBody: {
        HeaderMutation: {
            SetHeaders: [
                "x-selected-model: qwen14b-rack1",
                "x-gateway-destination-endpoint: 10.0.1.5:8000",
                "Authorization: Bearer sk-xxx",
                "content-length: 523",
                "traceparent: 00-abc123...",
            ],
            RemoveHeaders: ["content-length"],  // 旧的
        },
        BodyMutation: {
            Body: `{"model":"Qwen/Qwen2.5-14B-Instruct","messages":[...],"stream":true}`
        },
    }
}
```

注意 body 中的 `model` 被改写为上游实际模型名（`"auto"` → `"Qwen/Qwen2.5-14B-Instruct"`），header 中注入了鉴权、路由、追踪信息。Envoy 根据这些 header 将请求转发到正确的 vLLM 实例。

---

## 总结流程图

```
请求进入 → 保存原始body → Response API翻译? → 提取stream参数
  → 解析JSON → Looper检测? → 提取用户内容 → 决策评估(鉴权+分类+选模型)
  → Jailbreak检测 → PII检测 → 速率限制 → 缓存查询
  → RAG检索 → Modality路由(图像?) → Memory注入
  → 最终模型路由(改写body + 设置headers) → 返回给Envoy
```

每一步都可能短路返回（错误响应或缓存命中），正常流程会走完所有步骤后返回带有 body 和 header mutation 的 `ProcessingResponse`，Envoy 据此将修改后的请求转发给上游 LLM 服务。

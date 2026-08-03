# ExtProc 四阶段处理流水线详解

> 基于 `src/semantic-router/pkg/extproc/` 目录下的源码分析

---

## 总体架构

ExtProc 是一个 Envoy gRPC External Processor 服务，实现了 `ExternalProcessorServer` 接口。
Envoy 按照以下顺序将请求/响应的各阶段发送给 ExtProc 处理：

```
请求头(SEND) → 请求体(BUFFERED) → [Envoy 转发到上游] → 响应头(SEND) → 响应体(BUFFERED)
```

### 入口链路

1. **gRPC 服务注册**: `server.go:100` — `ext_proc.RegisterExternalProcessorServer(s.server, s.service)`
2. **请求委托**: `server.go:165-168` — `RouterService.Process()` 从 `atomic.Pointer` 加载当前 `OpenAIRouter` 并委托
3. **核心调度循环**: `processor_core.go:17-128` — `OpenAIRouter.Process()` 在 `for` 循环中用 `stream.Recv()` 接收消息，`switch` 分发到四个处理函数
4. **响应发送**: `utils.go:17-38` — `sendResponse()` 统一发送响应，处理 nil 安全

```go
// processor_core.go:68-107
switch v := req.Request.(type) {
case *ext_proc.ProcessingRequest_RequestHeaders:
    response, err := r.handleRequestHeaders(v, ctx)
case *ext_proc.ProcessingRequest_RequestBody:
    response, err := r.handleRequestBody(v, ctx)
case *ext_proc.ProcessingRequest_ResponseHeaders:
    response, err := r.handleResponseHeaders(v, ctx)
case *ext_proc.ProcessingRequest_ResponseBody:
    response, err := r.handleResponseBody(v, ctx)
}
```

### RequestContext 结构体
- **定义位置**: `processor_req_header.go:39-151`
- 在 `Process()` 循环开始时创建 (`processor_core.go:21-23`)
- 贯穿整个请求生命周期，存储所有中间状态

---

## 第一阶段：请求头处理 (Request Headers — SEND)

### 入口函数
- **`handleRequestHeaders()`** — `processor_req_header.go:154-286`

### 详细步骤

#### 1. 记录请求开始时间
```
processor_req_header.go:156 — ctx.StartTime = time.Now()
```

#### 2. 提取请求头到 map
```
processor_req_header.go:159-167 — 遍历 v.RequestHeaders.Headers.Headers，将所有 header 存入 headerMap
```
- 优先使用 `h.Value`，为空时回退到 `h.RawValue`

#### 3. 提取分布式追踪上下文
```
processor_req_header.go:170 — ctx.TraceContext = tracing.ExtractTraceContext(baseCtx, headerMap)
```
- 从 W3C traceparent/tracestate 头中恢复 OpenTelemetry context

#### 4. 启动根 Span
```
processor_req_header.go:173-176 — tracing.StartSpan(ctx.TraceContext, tracing.SpanRequestReceived)
```
- 创建 Server 类型的 span 用于追踪整个请求

#### 5. 存储所有请求头到 RequestContext
```
processor_req_header.go:179-201 — 遍历所有 header:
  - 存入 ctx.Headers map
  - 提取 x-request-id 存入 ctx.RequestID (processor_req_header.go:190-192)
  - 检测 x-vsr-looper-request 头，标记 ctx.LooperRequest (processor_req_header.go:194-198)
  - ext_authz 注入的认证头也会存入 ctx.Headers，供后续 CredentialResolver 使用
```

#### 6. 设置 Span 属性
```
processor_req_header.go:204-213 — 设置 request_id, http.method, http.path
```

#### 7. Router Replay API 拦截
```
processor_req_header.go:215-217 — r.handleRouterReplayAPI(method, path)
```
- **函数位置**: `router.go:779-844`
- 拦截 `GET /v1/router_replay` 和 `GET /v1/router_replay/{id}` 请求
- 如果匹配，直接返回 ImmediateResponse (短路)

#### 8. 流式响应检测
```
processor_req_header.go:220-225 — 检查 Accept: text/event-stream 头
```
- 如果包含 `text/event-stream`，设置 `ctx.ExpectStreamingResponse = true`

#### 9. GET /v1/models 请求处理
```
processor_req_header.go:228-231 — 拦截 GET /v1/models
```
- **函数位置**: `req_filter_models.go:27-80` — `handleModelsRequest()`
- 返回配置的模型列表（auto 模型 + 可选的后端模型）
- 直接返回 ImmediateResponse (短路)

#### 10. Response API 端点处理
```
processor_req_header.go:234-267 — 处理 /v1/responses 端点:
```
- `GET /v1/responses/{id}/input_items` → `ResponseAPIFilter.HandleGetInputItems()` (短路)
- `GET /v1/responses/{id}` → `ResponseAPIFilter.HandleGetResponse()` (短路)
- `DELETE /v1/responses/{id}` → `ResponseAPIFilter.HandleDeleteResponse()` (短路)
- `POST /v1/responses` → 标记 `ctx.ResponseAPICtx`，延迟到 body 阶段处理

#### 11. 构建并返回 CONTINUE 响应
```
processor_req_header.go:270-285 — 返回 HeadersResponse 带 CONTINUE 状态
```
- 不做任何 HeaderMutation (头修改在 body 阶段统一处理)
- 告诉 Envoy 继续发送请求体

---

## 第二阶段：请求体处理 (Request Body — BUFFERED)

### 入口函数
- **`handleRequestBody()`** — `processor_req_body.go:31-203`

### 详细步骤

#### 1. 记录路由处理开始时间
```
processor_req_body.go:33 — ctx.ProcessingStartTime = time.Now()
```

#### 2. 保存原始请求体
```
processor_req_body.go:35 — ctx.OriginalRequestBody = v.RequestBody.GetBody()
```

#### 3. Response API 请求体翻译
```
processor_req_body.go:38-56 — 如果是 /v1/responses POST:
  - 调用 r.ResponseAPIFilter.TranslateRequest() 将 Response API 格式转为 Chat Completions 格式
  - 更新 ctx.ResponseAPICtx 和 requestBody
```
- **函数位置**: `req_filter_response_api.go` — `ResponseAPIFilter.TranslateRequest()`

#### 4. 流式参数提取
```
processor_req_body.go:59-63 — extractStreamParam(requestBody)
```
- **函数位置**: `utils.go:50-62`
- 检查 JSON body 中的 `"stream": true` 参数
- 如果存在，设置 `ctx.ExpectStreamingResponse = true`

#### 5. 解析 OpenAI 请求
```
processor_req_body.go:66-74 — parseOpenAIRequest(requestBody)
```
- **函数位置**: `utils.go:41-47`
- 将 JSON 反序列化为 `openai.ChatCompletionNewParams`
- 失败时返回 gRPC InvalidArgument 错误

#### 6. 提取原始模型名
```
processor_req_body.go:77-82 — originalModel := openAIRequest.Model
```
- 设置 `ctx.RequestModel`

#### 7. Looper 内部请求检测
```
processor_req_body.go:86-89 — r.isLooperRequest(ctx)
```
- **函数位置**: `req_filter_looper.go:37`
- 如果是 Looper 内部请求，调用 `handleLooperInternalRequestWithPlugins()` 并直接返回

#### 8. 提取用户内容和非用户消息
```
processor_req_body.go:92-96 — extractUserAndNonUserContent(openAIRequest)
```
- **函数位置**: `utils.go:96-159`
- 遍历 messages 数组，分离 user/system/assistant 内容
- 存储 `ctx.UserContent` 用于后续幻觉检测

#### 9. Decision 评估 & 模型选择 (核心路由逻辑)
```
processor_req_body.go:100-106 — r.performDecisionEvaluation(originalModel, userContent, nonUserMessages, ctx)
```
- **函数位置**: `req_filter_classification.go:21-100+`
- 这是路由的核心，包含以下子步骤：

  **9a. 信号评估 (Signal Evaluation)**
  ```
  req_filter_classification.go:75 — r.Classifier.EvaluateAllSignalsWithHeaders(evaluationText, allMessagesText, ctx.Headers, false)
  ```
  评估所有信号类型：
  - **Keyword 信号**: 关键词匹配 → `ctx.VSRMatchedKeywords`
  - **Embedding 信号**: 语义嵌入相似度 → `ctx.VSRMatchedEmbeddings`
  - **Domain 信号**: 域分类 → `ctx.VSRMatchedDomains`
  - **Fact-check 信号**: 事实核查需求 → `ctx.VSRMatchedFactCheck`
  - **User Feedback 信号**: 用户反馈 → `ctx.VSRMatchedUserFeedback`
  - **Preference 信号**: 偏好 → `ctx.VSRMatchedPreference`
  - **Language 信号**: 语言检测 → `ctx.VSRMatchedLanguage`
  - **Context 信号**: 上下文（token 计数等） → `ctx.VSRMatchedContext`
  - **Complexity 信号**: 复杂度评估 → `ctx.VSRMatchedComplexity`
  - **Modality 信号**: 模态分类 (AR/DIFFUSION/BOTH) → `ctx.VSRMatchedModality`
  - **Authz 信号**: 授权评估 → `ctx.VSRMatchedAuthz`

  **9b. Fact-check 从信号中设置**
  ```
  req_filter_classification.go:100+ → r.setFactCheckFromSignals(ctx, signals.MatchedFactCheckRules)
  ```
  - **函数位置**: `req_filter_fact_check.go:15-45`
  - 设置 `ctx.FactCheckNeeded` 和 `ctx.FactCheckConfidence`
  - 调用 `r.checkRequestHasTools(ctx)` 检查请求中是否有 tool/RAG 上下文

  **9c. Decision Engine 评估**
  - 根据信号结果评估 Decision 规则 (AND/OR 组合)
  - 确定匹配的 Decision，选择模型

  **9d. Modality 分类**
  - 如果匹配了 Modality 信号，设置 `ctx.ModalityClassification`

  **9e. 模型选择**
  - 使用 ModelSelector Registry 根据 Decision 算法选择模型
  - 算法类型: static, elo, router_dc, automix, hybrid, knn, kmeans, svm, mlp
  - 设置 `ctx.VSRSelectedModel`, `ctx.VSRSelectionMethod`

  **9f. 授权错误传播**
  - 如果 authz 信号评估失败，返回 403 错误

  返回值: `(decisionName, confidence, reasoningDecision, selectedModel, authzErr)`

#### 10. 记录请求指标
```
processor_req_body.go:109 — metrics.RecordModelRequest(selectedModel)
```

#### 11. Jailbreak 检测
```
processor_req_body.go:112-117 — r.performJailbreaks(ctx, userContent, nonUserMessages, decisionName)
```
- **函数位置**: `req_filter_jailbreak.go:17-116`
- 检查是否为该 Decision 启用了 jailbreak 检测
- 获取 Decision 特定的阈值和 include_history 设置
- 调用 `r.Classifier.AnalyzeContentForJailbreakWithThreshold()`
- 如果检测到 jailbreak → 设置 `ctx.JailbreakDetected`, 记录安全日志, 返回 403 (短路)
- 启动 Router Replay 记录被拦截的请求

#### 12. PII 检测
```
processor_req_body.go:120-127 — r.performPIIDetection(ctx, userContent, nonUserMessages, decisionName)
```
- **函数位置**: `req_filter_pii.go:18-144`
- 子步骤:
  - `isPIIDetectionEnabled()` — 检查 Decision 是否启用 PII 插件 (`req_filter_pii.go:39-61`)
  - `detectPIIWithTracing()` — PII 实体检测，带 tracing (`req_filter_pii.go:65-114`)
  - `checkPIIPolicy()` — 检查 PII 策略是否允许 (`req_filter_pii.go:117-144`)
- 如果 PII 策略违规 → 设置 `ctx.PIIBlocked`, 返回 403 (短路)

#### 13. 速率限制检查
```
processor_req_body.go:131-145 — r.RateLimiter.Check(rlCtx)
```
- `buildRateLimitContext()` — 构建限流上下文 (`processor_req_body.go:1106`)
- 检查 RPM/TPM 配额
- 如果被限流 → `createRateLimitResponse()` 返回 429 (短路)
- 存储 `ctx.RateLimitCtx` 供响应阶段汇报 token 使用

#### 14. 缓存处理
```
processor_req_body.go:148-152 — r.handleCaching(ctx, decisionName)
```
- **函数位置**: `req_filter_cache.go:17-148`
- 子步骤:
  - `cache.ExtractQueryFromOpenAIRequest()` — 提取模型和查询文本
  - `r.Config.IsCacheEnabledForDecision()` — 检查 Decision 是否启用缓存
  - `r.Config.GetCacheSimilarityThresholdForDecision()` — 获取 Decision 特定的相似度阈值
  - `r.Cache.FindSimilarWithThreshold()` — 语义缓存查找
  - 如果缓存命中 → 创建 `ImmediateResponse` 直接返回缓存内容 (短路)
  - 如果缓存未命中 → `r.Cache.AddPendingRequest()` 添加待缓存请求
- Looper 请求跳过缓存读取但仍然写入

#### 15. RAG 插件执行
```
processor_req_body.go:156-159 — r.executeRAGPlugin(ctx, decisionName)
```
- **函数位置**: `req_filter_rag.go:20-60+`
- 检查 Decision 是否配置了 RAG 插件
- 根据后端类型分发:
  - **milvus**: `req_filter_rag_milvus.go`
  - **external_api**: `req_filter_rag_external.go`
  - **openai_file_search**: `req_filter_rag_openai.go`
  - **openai_vectorstore**: `req_filter_rag_vectorstore.go`
  - **mcp**: `req_filter_rag_mcp.go`
  - **hybrid**: `req_filter_rag_hybrid.go`
  - **cache**: `req_filter_rag_cache.go`
- 检索到的上下文存入 `ctx.RAGRetrievedContext`
- 将 RAG 上下文注入到请求的 messages 中
- 如果失败且 `on_failure=block` → 返回 503 (短路)

#### 16. Modality 路由
```
processor_req_body.go:164-186 — r.handleModalityFromDecision(ctx, openAIRequest)
```
- **函数位置**: `req_filter_modality.go:160`
- 如果 Response API 请求包含 `image_generation` tool → 强制设置 modality
- 如果 modality 是 DIFFUSION 或 BOTH → 执行图片生成并短路返回

#### 17. Memory 检索
```
processor_req_body.go:191-199 — r.handleMemoryRetrieval(ctx, userContent, requestBody, openAIRequest)
```
- **函数位置**: `processor_req_body.go:943`
- 从 Milvus memory store 检索相关记忆
- 将记忆上下文注入到请求中
- 存储 `ctx.MemoryContext`

#### 18. 模型路由 (最终阶段)
```
processor_req_body.go:202 — r.handleModelRouting(openAIRequest, originalModel, decisionName, reasoningDecision, selectedModel, ctx)
```
- **函数位置**: `processor_req_body.go:207-245`
- 根据条件分支:

  **18a. Anthropic 路由**
  ```
  processor_req_body.go:226-228 — r.handleAnthropicRouting()
  ```
  - **函数位置**: `processor_req_body.go:250-340`
  - 将 OpenAI 格式转换为 Anthropic 格式: `anthropic.ToAnthropicRequestBody()`
  - 设置 `ctx.APIFormat = "anthropic"`
  - 解析 API Key: `r.CredentialResolver.KeyForProvider(authz.ProviderAnthropic, ...)`
  - 构建 Anthropic 专用 header
  - 设置 `ClearRouteCache = true`
  - 返回 BodyResponse 带 HeaderMutation + BodyMutation

  **18b. 指定模型路由 (非 auto)**
  ```
  processor_req_body.go:233 — r.handleSpecifiedModelRouting()
  ```
  - **函数位置**: `processor_req_body.go:408-447`
  - 选择端点: `selectEndpointForModel()` (`processor_req_body.go:452-468`)
  - 解析模型名到实际上游名称: `resolveModelNameForEndpoint()` (`processor_req_body.go:473-482`)
  - 创建路由响应: `createSpecifiedModelResponse()` (`processor_req_body.go:737`)
    - 启动上游 Span: `startUpstreamSpanAndInjectHeaders()` (`processor_req_body.go:529`)
    - 解析 Provider Profile 和认证: `resolveProviderAuth()` (`processor_req_body.go:566`)
    - 通过 CredentialResolver 获取 API Key
    - 设置 header: `x-gateway-destination-endpoint`, `x-selected-model`, Authorization, trace context
    - 应用 Decision 的 header_mutation 插件: `buildHeaderMutations()` (`req_filter_header_mutation.go:12-58`)
  - Tool 选择: `handleToolSelectionForRequest()` (`req_filter_tools.go:24-31`)
  - 记录路由延迟: `recordRoutingLatency()` (`recorder.go:86-89`)

  **18c. Looper 执行**
  ```
  processor_req_body.go:234-237 — r.handleLooperExecution()
  ```
  - **函数位置**: `req_filter_looper.go`
  - 当 Decision 配置了 looper 算法时触发

  **18d. Auto 模型路由**
  ```
  processor_req_body.go:238-239 — r.handleAutoModelRouting()
  ```
  - **函数位置**: `processor_req_body.go:343-405`
  - 子步骤:
    - 记录路由决策: `recordRoutingDecision()` (`recorder.go:37-61`)
    - 追踪 VSR 决策: `trackVSRDecision()` (`recorder.go:66-75`)
    - 记录路由指标: `metrics.RecordModelRouting()`
    - 选择端点: `selectEndpointForModel()` (`processor_req_body.go:452`)
    - 解析上游模型名: `resolveModelNameForEndpoint()` (`processor_req_body.go:473`)
    - 修改请求体: `modifyRequestBodyForAutoRouting()` (`processor_req_body.go:485`)
      - 修改 model 字段
      - 序列化 (保留 stream 参数): `serializeOpenAIRequestWithStream()` (`utils.go:65-93`)
      - 设置推理模式: `setReasoningModeToRequestBody()` (`req_filter_reason.go:14-147`)
      - 注入系统提示: `addSystemPromptIfConfigured()` (`req_filter_sys_prompt.go:18-80`)
      - 注入 Memory 上下文: `injectSystemMessage()` (processor_req_body.go:517)
    - 创建路由响应: `createRoutingResponse()` (`processor_req_body.go:589-731`)
      - 设置 BodyMutation (修改后的请求体)
      - 移除旧 content-length，设置新 content-length
      - 启动上游 Span + 注入 trace context headers
      - 解析 Provider Profile 和认证
      - 设置路由 headers: endpoint, model, auth, path
      - 应用 Decision 的 header_mutation 插件
      - 应用 Provider Profile 的 extra_headers
      - 剥离 ext_authz 注入的敏感 headers
    - 路由缓存清除: `setClearRouteCache()` (`recorder.go:78-83`)
    - Router Replay 捕获: `startRouterReplay()` (`recorder.go:93-217`)
    - Tool 选择: `handleToolSelectionForRequest()` (`req_filter_tools.go:24-31`)
      - 检查 tool_choice="auto"
      - 语义相似度匹配: `ToolsDatabase.FindSimilarTools()`
      - 高级过滤 (可选): `tools.FilterAndRankTools()`
      - 更新请求体: `updateRequestWithTools()` (`req_filter_tools.go:142-235`)
    - 记录路由延迟

### Body 阶段返回的 ProcessingResponse 结构
```
ProcessingResponse_RequestBody {
  BodyResponse {
    CommonResponse {
      Status: CONTINUE
      HeaderMutation: {
        SetHeaders: [content-length, x-gateway-destination-endpoint, x-selected-model,
                     Authorization, traceparent, tracestate, ...]
        RemoveHeaders: [content-length, x-user-openai-key, x-user-anthropic-key, ...]
      }
      BodyMutation: {
        Body: <修改后的 JSON 请求体>
      }
      ClearRouteCache: true/false
    }
  }
}
```

---

## 第三阶段：响应头处理 (Response Headers — SEND)

### 入口函数
- **`handleResponseHeaders()`** — `processor_res_header.go:22-333`

### 详细步骤

#### 1. Looper 请求快速路径
```
processor_res_header.go:24-43 — 如果 ctx.LooperRequest:
  - 提取 status code
  - 更新 Router Replay 状态
  - 返回 CONTINUE (不做任何处理)
```

#### 2. 检测上游 HTTP 状态码
```
processor_res_header.go:49-63 — getStatusFromHeaders()
```
- **函数位置**: `processor_res_header.go:336-355`
- 从 `:status` 伪头中提取状态码
- 检测流式响应: `isStreamingContentType()` (`processor_res_header.go:365-381`)
  - 检查 Content-Type 是否包含 `text/event-stream`
  - 设置 `ctx.IsStreamingResponse`
- 记录错误指标:
  - 5xx → `metrics.RecordRequestError(model, "upstream_5xx")`
  - 4xx → `metrics.RecordRequestError(model, "upstream_4xx")`

#### 3. 结束上游请求 Span
```
processor_res_header.go:66-78 — 结束在 body 阶段启动的 ctx.UpstreamSpan
```
- 添加 `http.status_code` 属性
- 如果非 2xx → 标记 span 为 Error

#### 4. 非流式 TTFT 记录
```
processor_res_header.go:83-92 — 对于非流式响应，记录 TTFT
```
- `metrics.RecordModelTTFT(ctx.RequestModel, ttft)`
- `latency.UpdateTTFT(ctx.RequestModel, ttft)` — 更新延迟缓存用于 latency_aware 选择

#### 5. 更新 Router Replay 状态
```
processor_res_header.go:95 — r.updateRouterReplayStatus(ctx, statusCode, ctx.IsStreamingResponse)
```
- **函数位置**: `recorder.go:220-237`

#### 6. 构建 VSR 决策追踪 Headers
```
processor_res_header.go:98-309 — 如果请求成功 (2xx) 且非缓存命中:
```
注入以下响应头:
| Header | 值 | 代码位置 |
|--------|------|----------|
| `x-vsr-selected-category` | 域分类类别 | `processor_res_header.go:106-112` |
| `x-vsr-selected-decision` | Decision 名称 | `processor_res_header.go:115-122` |
| `x-vsr-selected-confidence` | 决策置信度 | `processor_res_header.go:125-132` |
| `x-vsr-selected-modality` | 模态分类 (AR/DIFFUSION) | `processor_res_header.go:135-146` |
| `x-vsr-matched-keywords` | 匹配的关键词规则 | `processor_res_header.go:149-156` |
| `x-vsr-selected-reasoning` | 推理模式 on/off | `processor_res_header.go:159-166` |
| `x-vsr-selected-model` | 选择的模型 | `processor_res_header.go:169-176` |
| `x-vsr-injected-system-prompt` | true/false | `processor_res_header.go:179-188` |
| `x-vsr-matched-embeddings` | 匹配的嵌入规则 | `processor_res_header.go:200-207` |
| `x-vsr-matched-domains` | 匹配的域规则 | `processor_res_header.go:209-216` |
| `x-vsr-matched-fact-check` | 事实核查信号 | `processor_res_header.go:218-225` |
| `x-vsr-matched-user-feedback` | 用户反馈信号 | `processor_res_header.go:227-234` |
| `x-vsr-matched-preference` | 偏好信号 | `processor_res_header.go:236-243` |
| `x-vsr-matched-language` | 语言信号 | `processor_res_header.go:245-252` |
| `x-vsr-matched-context` | 上下文信号 | `processor_res_header.go:255-262` |
| `x-vsr-context-token-count` | token 计数 | `processor_res_header.go:265-272` |
| `x-vsr-matched-complexity` | 复杂度信号 | `processor_res_header.go:275-282` |
| `x-vsr-matched-authz` | 授权信号 | `processor_res_header.go:285-292` |
| `x-router-replay-id` | Replay 记录 ID | `processor_res_header.go:295-302` |

#### 7. 构建响应
```
processor_res_header.go:313-322 — 返回 ResponseHeaders 带 CONTINUE + HeaderMutation
```

#### 8. 流式响应模式覆盖
```
processor_res_header.go:326-330 — 如果是流式响应:
  response.ModeOverride = ProcessingMode{ResponseBodyMode: STREAMED}
```
- 指示 Envoy 以 STREAMED 模式发送响应体 (而非 BUFFERED)
- 需要 Envoy 配置 `allow_mode_override: true`

---

## 第四阶段：响应体处理 (Response Body — BUFFERED)

### 入口函数
- **`handleResponseBody()`** — `processor_res_body.go:23-374`

### 详细步骤

#### 1. 计算完成延迟
```
processor_res_body.go:24 — completionLatency := time.Since(ctx.StartTime)
```

#### 2. 递减活跃请求计数
```
processor_res_body.go:27 — defer metrics.DecrementModelActiveRequests(ctx.RequestModel)
```

#### 3. Looper 请求快速路径
```
processor_res_body.go:31-47 — 如果 ctx.LooperRequest:
  - 捕获 response body 到 Router Replay
  - 返回 CONTINUE
```

#### 4. Anthropic 响应转换
```
processor_res_body.go:52-65 — 如果 ctx.APIFormat == "anthropic":
  - anthropic.ToOpenAIResponseBody() 将 Anthropic 格式转为 OpenAI 格式
  - 标记 anthropicTransformed = true
```

#### 5. 流式响应处理
```
processor_res_body.go:69-128 — 如果 ctx.IsStreamingResponse:
```
  **5a. 首块 TTFT 记录**
  ```
  processor_res_body.go:70-80 — 第一个 body chunk 到达时记录 TTFT
  ```
  - `metrics.RecordModelTTFT()`, `latency.UpdateTTFT()`

  **5b. 累积流式 chunks**
  ```
  processor_res_body.go:83-88 — 将 chunk 添加到 ctx.StreamingChunks
  ```

  **5c. 解析 SSE chunk**
  ```
  processor_res_body.go:91 — r.parseStreamingChunk(chunk, ctx)
  ```
  - **函数位置**: `processor_res_body.go:377-431`
  - 解析 `data: {...}` 格式的 SSE 事件
  - 提取元数据 (id, model, created) 从首个 chunk
  - 累积 `delta.content` 到 `ctx.StreamingContent`
  - 提取 `finish_reason` 和 `usage`

  **5d. 检查 [DONE] 标记**
  ```
  processor_res_body.go:94-115 — 如果 chunk 包含 "data: [DONE]":
  ```
  - 标记 `ctx.StreamingComplete = true`
  - 记录完成延迟指标
  - 重建并缓存完整响应: `cacheStreamingResponse()` (`processor_res_body.go:434-586`)
    - 5 项安全检查 (complete, not aborted, has content, has metadata, valid structure)
    - 重建 `openai.ChatCompletion` JSON
    - 汇报 token 使用到限流器
    - 记录 token 指标和 TPOT
    - 写入缓存: `Cache.AddEntry()` 或 `Cache.UpdateWithResponse()`
  - 附加 replay 响应: `attachRouterReplayResponse()`

  **5e. 返回 CONTINUE (流式)**
  ```
  processor_res_body.go:118-127 — 返回 CONTINUE，让 Envoy 继续转发 chunk
  ```

#### 6. 非流式响应：解析 token 使用
```
processor_res_body.go:131-137 — json.Unmarshal → openai.ChatCompletion
```
- 提取 `PromptTokens`, `CompletionTokens`

#### 7. 汇报 token 使用到限流器
```
processor_res_body.go:140-146 — r.RateLimiter.Report()
```
- 提交 InputTokens, OutputTokens, TotalTokens

#### 8. 记录 token 和延迟指标
```
processor_res_body.go:149-212 — 如果有模型名:
```
- `metrics.RecordModelTokensDetailed()` — prompt + completion tokens
- `metrics.RecordModelCompletionLatency()` — 完成延迟
- `metrics.RecordModelTPOT()` + `latency.UpdateTPOT()` — 每 token 时间
- `metrics.RecordModelWindowedRequest()` — 窗口化指标 (用于负载均衡)
- 计算并记录费用: `metrics.RecordModelCost()` (如果配置了定价)
- 结构化日志: `logging.LogEvent("llm_usage", ...)`

#### 9. 更新缓存
```
processor_res_body.go:215-228 — r.Cache.UpdateWithResponse(ctx.RequestID, responseBody, ttlSeconds)
```
- 使用 Decision 特定的 TTL

#### 10. Memory 提取 (异步)
```
processor_res_body.go:232-278 — 如果启用了 Memory Extractor + auto_store:
```
- 在后台 goroutine 中执行，不增加响应延迟
- 提取当前用户消息和助手响应
- 调用 `r.MemoryExtractor.ProcessResponse()` 提取并存储记忆

#### 11. Response API 响应翻译
```
processor_res_body.go:281-291 — 如果是 Response API 请求:
  - r.ResponseAPIFilter.TranslateResponse() 将 ChatCompletion 格式转为 Response API 格式
```

#### 12. 构建 Body Mutation
```
processor_res_body.go:294-308 — 如果响应被转换 (Anthropic 或 Response API):
  - 设置 BodyMutation 替换响应体
  - 移除 content-length 让 Envoy 重新计算
```

#### 13. 幻觉检测
```
processor_res_body.go:311-313 — r.performHallucinationDetection(ctx, responseBody)
```
- **函数位置**: `res_filter_hallucination.go:21-96`
- 条件检查: `shouldPerformHallucinationDetection()` (`req_filter_fact_check.go:119-148`)
  - 必须有分类器且启用幻觉检测
  - Decision 的 hallucination 插件必须启用
  - 必须 `ctx.FactCheckNeeded == true`
  - 必须有 tool/RAG 上下文 (`ctx.HasToolsForFactCheck && ctx.ToolResultsContext != ""`)
- 提取助手内容: `extractAssistantContentFromResponse()` (`res_filter_hallucination.go:172-191`)
- 检查 NLI 是否启用: `isNLIEnabledForDecision()` (`res_filter_hallucination.go:194-209`)
- 基础检测: `r.Classifier.DetectHallucination()` 或 NLI 增强检测: `DetectHallucinationWithNLI()`
- 设置 `ctx.HallucinationDetected`, `ctx.HallucinationSpans`, `ctx.HallucinationConfidence`
- 如果 action="block" → 返回错误响应 (短路)

#### 14. 检查未验证事实响应
```
processor_res_body.go:317-322 — r.checkUnverifiedFactualResponse(ctx)
```
- **函数位置**: `res_filter_hallucination.go:412-423`
- 当 `FactCheckNeeded=true` 但没有 tool 上下文时 → 标记 `ctx.UnverifiedFactualResponse`

#### 15. 应用幻觉警告
```
processor_res_body.go:342-347 — r.applyHallucinationWarning(response, ctx, modifiedBody)
```
- **函数位置**: `res_filter_hallucination.go:213-233`
- 根据 action 配置:
  - **"header"**: 添加 `x-hallucination-detected: true`, `x-hallucination-spans` 等响应头 (`res_filter_hallucination.go:250-303`)
  - **"body"**: 在响应内容前添加警告文本 (`res_filter_hallucination.go:306-348`)
  - **"none"**: 不做任何处理

#### 16. 应用未验证事实警告
```
processor_res_body.go:350-355 — r.applyUnverifiedFactualWarning(response, ctx, modifiedBody)
```
- **函数位置**: `res_filter_hallucination.go:427-446`
- 类似幻觉警告，添加 `x-unverified-factual-response`, `x-fact-check-needed`, `x-verification-context-missing` 头

#### 17. 更新 Body Mutation (如果 body 被修改)
```
processor_res_body.go:358-365 — 如果 body 被幻觉/事实警告修改:
  - 更新 BodyMutation
```

#### 18. 更新 Router Replay 幻觉状态
```
processor_res_body.go:368 — r.updateRouterReplayHallucinationStatus(ctx)
```
- **函数位置**: `recorder.go:268-299`
- 记录幻觉检测结果到 replay 记录

#### 19. 捕获 Replay 响应
```
processor_res_body.go:371 — r.attachRouterReplayResponse(ctx, finalBody, true)
```
- **函数位置**: `recorder.go:240-265`
- 存储响应体到 replay 记录 (如果配置了 capture_response_body)
- 记录 `router_replay_complete` 日志

#### 20. 返回最终响应
```
processor_res_body.go:373 — return response, nil
```

### Body 阶段返回的 ProcessingResponse 结构
```
ProcessingResponse_ResponseBody {
  BodyResponse {
    CommonResponse {
      Status: CONTINUE
      HeaderMutation: {   // 可选: 幻觉/事实检查警告 headers
        SetHeaders: [x-hallucination-detected, x-hallucination-spans, ...]
      }
      BodyMutation: {     // 可选: 如果响应被转换或添加了警告
        Body: <修改后的响应体>
      }
    }
  }
}
```

---

## 文件索引

| 文件 | 职责 |
|------|------|
| `server.go` | gRPC Server, RouterService (atomic swap), 配置热加载 |
| `processor_core.go` | Process() 主循环, 请求调度 |
| `processor_req_header.go` | 请求头处理, RequestContext 定义 |
| `processor_req_body.go` | 请求体处理, 模型路由, 响应构建 |
| `processor_res_header.go` | 响应头处理, VSR 追踪 headers 注入 |
| `processor_res_body.go` | 响应体处理, 缓存更新, 指标记录 |
| `router.go` | OpenAIRouter 结构体, NewOpenAIRouter(), 初始化 |
| `utils.go` | 工具函数: sendResponse, parseOpenAIRequest, extractStreamParam |
| `recorder.go` | 路由记录: 日志, tracing, VSR 追踪, Router Replay |
| `req_filter_classification.go` | Decision 评估, 信号评估 |
| `req_filter_cache.go` | 语义缓存查找/存储 |
| `req_filter_jailbreak.go` | Jailbreak 检测 |
| `req_filter_pii.go` | PII 检测与策略检查 |
| `req_filter_fact_check.go` | 事实核查信号, 幻觉检测条件 |
| `req_filter_reason.go` | 推理模式设置 |
| `req_filter_sys_prompt.go` | 系统提示注入 |
| `req_filter_tools.go` | 自动 Tool 选择 |
| `req_filter_header_mutation.go` | Decision header mutation 插件 |
| `req_filter_modality.go` | 模态路由 (AR/DIFFUSION) |
| `req_filter_looper.go` | Looper 内部请求处理 |
| `req_filter_response_api.go` | Response API 翻译 |
| `req_filter_rag.go` | RAG 插件入口 |
| `req_filter_rag_milvus.go` | Milvus RAG 后端 |
| `req_filter_rag_external.go` | 外部 API RAG 后端 |
| `req_filter_rag_openai.go` | OpenAI File Search RAG 后端 |
| `req_filter_rag_mcp.go` | MCP RAG 后端 |
| `req_filter_rag_hybrid.go` | 混合 RAG 后端 |
| `req_filter_rag_cache.go` | 缓存 RAG 后端 |
| `req_filter_rag_vectorstore.go` | OpenAI VectorStore RAG 后端 |
| `res_filter_hallucination.go` | 幻觉检测与警告 |

---

## 流程图

```
客户端请求
    │
    ▼
┌─────────────────────────────────────────────┐
│  第一阶段: handleRequestHeaders()            │
│  processor_req_header.go:154-286             │
│                                             │
│  1. 记录开始时间                              │
│  2. 提取 headers → ctx.Headers               │
│  3. 提取 trace context                       │
│  4. 启动 root span                           │
│  5. 存储 request-id, looper 标志              │
│  6. 拦截: /v1/router_replay (短路)            │
│  7. 检测流式请求 (Accept header)              │
│  8. 拦截: GET /v1/models (短路)               │
│  9. 拦截: Response API GET/DELETE (短路)      │
│ 10. POST /v1/responses → 标记                 │
│                                             │
│  返回: CONTINUE (无 header 修改)              │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  第二阶段: handleRequestBody()               │
│  processor_req_body.go:31-203                │
│                                             │
│  1. 保存原始请求体                             │
│  2. Response API 翻译 (如需)                  │
│  3. 流式参数检测                              │
│  4. 解析 OpenAI 请求                          │
│  5. Looper 检测 (短路)                        │
│  6. 提取用户内容                              │
│  7. ★ Decision 评估 (信号→Decision→模型选择)   │
│  8. Jailbreak 检测 → 403 (短路)              │
│  9. PII 检测 → 403 (短路)                    │
│ 10. 速率限制 → 429 (短路)                     │
│ 11. 缓存查找 → ImmediateResponse (短路)       │
│ 12. RAG 检索 + 上下文注入                     │
│ 13. Modality 路由 → 图片生成 (短路)           │
│ 14. Memory 检索 + 注入                        │
│ 15. ★ 模型路由: header + body mutation        │
│     - 推理模式设置                             │
│     - 系统提示注入                             │
│     - 端点选择 + API key                      │
│     - Tool 选择                               │
│     - Router Replay 开始                      │
│                                             │
│  返回: CONTINUE + HeaderMutation + BodyMutation│
└─────────────────┬───────────────────────────┘
                  │
            [Envoy 转发到上游 vLLM/Anthropic/...]
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  第三阶段: handleResponseHeaders()           │
│  processor_res_header.go:22-333              │
│                                             │
│  1. Looper 快速路径 (CONTINUE)               │
│  2. 提取 :status, 检测流式 Content-Type       │
│  3. 记录 4xx/5xx 错误指标                     │
│  4. 结束上游 Span                            │
│  5. 非流式: 记录 TTFT                        │
│  6. 更新 Router Replay 状态                   │
│  7. ★ 注入 VSR 追踪 headers (20+ 个)         │
│  8. 流式: 设置 ModeOverride=STREAMED          │
│                                             │
│  返回: CONTINUE + HeaderMutation (VSR headers)│
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  第四阶段: handleResponseBody()              │
│  processor_res_body.go:23-374                │
│                                             │
│  1. 计算完成延迟                              │
│  2. Looper 快速路径 (CONTINUE)               │
│  3. Anthropic → OpenAI 响应转换               │
│  4. 流式处理:                                 │
│     - 首块 TTFT                               │
│     - 累积 chunks                             │
│     - [DONE] → 重建 + 缓存                    │
│  5. 非流式: 解析 token 使用                    │
│  6. 汇报 token 到限流器                       │
│  7. ★ 记录 token/延迟/TPOT/费用指标           │
│  8. 更新语义缓存                              │
│  9. Memory 提取 (异步 goroutine)              │
│ 10. Response API 响应翻译                     │
│ 11. ★ 幻觉检测 (基础 or NLI 增强)            │
│ 12. 未验证事实检查                             │
│ 13. 应用幻觉/事实警告 (header/body/block)     │
│ 14. 更新 Router Replay 幻觉状态               │
│ 15. 附加 Replay 响应                          │
│                                             │
│  返回: CONTINUE + 可选 HeaderMutation/BodyMutation│
└─────────────────────────────────────────────┘
                  │
                  ▼
            客户端接收响应
```

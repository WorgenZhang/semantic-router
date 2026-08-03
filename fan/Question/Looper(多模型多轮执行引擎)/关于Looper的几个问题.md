## 完整解答

### 一、什么时候 `ctx.LooperRequest = true`？

**触发条件只有一个**：在 `processor_req_header.go:194` 中，当收到的请求头包含 `x-vsr-looper-request: true` 时：

```go
if h.Key == headers.VSRLooperRequest && headerValue == "true" {
    ctx.LooperRequest = true
}
```

那么**谁会发送带这个 header 的请求**？答案是 **Looper Client**（`looper/client.go:163`）：

```go
httpReq.Header.Set("x-vsr-looper-request", "true")
httpReq.Header.Set("x-vsr-looper-iteration", fmt.Sprintf("%d", iteration))
```

---

### 二、什么是"Router 内部发起的重放请求"？

这需要理解 **Looper 系统的架构**。它的核心思想是：**一个用户请求，调多个模型，然后用算法选出最好的回答**。

#### 具体场景举例：

---

#### 场景1：Confidence（自信度）算法

**用户请求**：`"请解释量子纠缠"`

**流程**：
```
用户 → Envoy → ExtProc（handleRequestBody）
                    ↓
              检测到 decision 配置了 algorithm: confidence
              调用 handleLooperExecution()
                    ↓
              Looper Client 发出内部请求:
              
              【第1轮】→ 发给 llama-3-8b（小模型，便宜）
                        请求头: x-vsr-looper-request: true
                                x-vsr-looper-iteration: 1
                        ← 收到回答 + logprobs
                        → 计算置信度 = 0.92（很自信）
                        → 置信度 > 阈值 0.85，停止！用这个回答
              
              或者：
              
              【第1轮】→ 发给 llama-3-8b
                        ← 置信度 = 0.45（不自信）
              【第2轮】→ 发给 llama-3-70b（大模型，贵）
                        请求头: x-vsr-looper-request: true
                                x-vsr-looper-iteration: 2
                        ← 收到回答
                        → 用大模型的回答
```

**目的**：简单问题用小模型省钱，难问题自动升级到大模型。

---

#### 场景2：Ratings（评分）算法

**用户请求**：`"写一首关于春天的诗"`

**流程**：
```
Looper 同时/依次调多个模型：
  【内部请求1】→ model-A (x-vsr-looper-request: true)
  【内部请求2】→ model-B (x-vsr-looper-request: true)
  【内部请求3】→ model-C (x-vsr-looper-request: true)
  
  收集所有回答后，用评分算法选出最好的那个返回给用户
```

---

#### 场景3：ReMoM（Reasoning + Mixture of Models）算法

**用户请求**：`"设计一个分布式缓存系统"`

**流程**：
```
【第1轮】→ reasoning-model 做推理分析（内部请求，x-vsr-looper-request: true）
【第2轮】→ 根据推理结果选 coding-model 生成代码（内部请求）
【第3轮】→ 可能还有验证轮
```

---

#### 场景4：RL-Driven（强化学习驱动）算法

根据历史反馈数据，动态学习哪个模型在哪类问题上表现最好，然后自动尝试。

---

### 三、为什么叫"重放"？

因为从 Envoy 的视角看，**这些内部请求和正常用户请求走的是同一条路径**：

```
                   ┌─────────────────────────────────────┐
  用户请求 ───→    │     Envoy + ExtProc                  │  ───→ LLM Backend
                   │                                     │
  Looper内部请求 →  │  （同一个入口，同一套 ExtProc 处理链）  │  ───→ LLM Backend
                   └─────────────────────────────────────┘
```

Looper 不直接调 LLM，而是**把请求重新发回 Envoy**（`looper.Endpoint` 配置的就是 Envoy 入口地址）。这样的好处是：
- 复用 Envoy 的负载均衡、路由规则
- 复用 ExtProc 的 header 注入（如 Authorization）
- 统一的请求入口和监控

所以这些请求像是在"重放"用户的请求，只是换了不同的目标模型。

---

### 四、为什么 Looper 内部请求可以"直接放行"（只返回 CONTINUE）？

这是最关键的问题。原因有**三个**：

#### 原因1：防止无限递归

```
用户请求 → ExtProc 处理（分类、RAG、缓存...） → 触发 Looper
  → Looper 发内部请求 → 又进 ExtProc
      → 如果又做分类、RAG... → 又触发 Looper → 无限循环！
```

所以内部请求必须标记为 `LooperRequest`，让 ExtProc **跳过所有插件处理**（分类、RAG、缓存、记忆等），直接路由到目标模型。

#### 原因2：响应头无需附加 VSR 决策 header

那些 `x-vsr-selected-model`、`x-vsr-selected-category` 等 header 是给**最终客户端**看的。但 Looper 内部请求的响应不是给客户端的——它是给 Looper 算法消费的：

```
Looper 发内部请求 → LLM 回复 → 响应经过 ExtProc 
                                    ↓
                              如果加了 x-vsr-* headers
                              Looper 收到后根本不需要这些
                              白白浪费处理时间

Looper 收到原始响应 → 用算法判断 → 最终结果通过 handleLooperExecution 
                                   直接返回给用户（带完整 headers）
```

#### 原因3：指标不应重复计算

```
用户发 1 个请求 → Looper 可能发 3 个内部请求

如果每个内部请求都记录 TTFT、都做 span tracing：
  - TTFT 被记录 3 次（但用户只感知到 1 次延迟）
  - 错误计数被放大 3 倍
  - dashboard 数据失真
```

所以内部请求的响应头处理只做最小操作：
1. 提取状态码（200/500/...） ← Looper 需要知道请求是否成功
2. 更新 replay 状态 ← 记录每个子请求的结果，供事后分析
3. 返回 CONTINUE ← 让响应正常回到 Looper Client

**最终的指标记录、span 结束、VSR headers 注入，都在外层的正常请求（非 Looper 请求）那条路径上完成。**

---

### 五、完整对比图

```
┌──────────────────────────────────────────────────────────────────┐
│                    正常用户请求的 handleResponseHeaders            │
├──────────────────────────────────────────────────────────────────┤
│ 1. 提取状态码                                                     │
│ 2. 记录错误指标（4xx/5xx）                                        │
│ 3. 结束 UpstreamSpan（OpenTelemetry）                             │
│ 4. 记录 TTFT（延迟指标）                                          │
│ 5. 更新 replay 状态                                              │
│ 6. 注入 10+ 个 x-vsr-* 响应头（给客户端看）                       │
│ 7. 设置 STREAMED mode（如果是 SSE）                               │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                   Looper 内部请求的 handleResponseHeaders          │
├──────────────────────────────────────────────────────────────────┤
│ 1. 提取状态码                                                     │
│ 2. 更新 replay 状态（记录子请求结果）                              │
│ 3. 返回 CONTINUE（放行，不加任何额外处理）                         │
└──────────────────────────────────────────────────────────────────┘
```

简单说：**Looper 内部请求是"工具人"，它的响应是中间数据，不是最终产品，所以不需要包装和装饰，直接原样交给 Looper 算法就行。**

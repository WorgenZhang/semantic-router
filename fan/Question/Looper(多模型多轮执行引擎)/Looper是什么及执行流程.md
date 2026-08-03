## Looper 是什么？

**Looper 是一个「多模型多轮执行引擎」**。当 semantic-router 判断一个请求需要「比单模型更好的结果」时，它不会只调用一个模型，而是通过 Looper 调用多个模型，并对结果进行聚合/升级/综合，最终返回一个更高质量的答案给用户。

用户完全感知不到这个过程 —— 对用户来说还是一次普通的 API 调用，但背后可能经历了多次模型调用。

---

## 为什么需要「Looper 内部请求检测」？

Looper 执行时，会向 **同一个 Envoy 入口** 发送 HTTP 请求（通过 `client.go` 中的 `CallModel`）。这些请求会再次经过 ExtProc（即 `handleRequestBody`）。

如果不做检测，这些内部请求会被当作普通用户请求，再次走完整的 16 步流程（鉴权、安全检测、速率限制、RAG、Memory 注入等），导致：
1. **无限递归**：Looper 请求触发 Looper → 再触发 Looper → 死循环
2. **重复安全检测**：原始请求已经通过了 jailbreak/PII 检测，内部请求不需要重复
3. **速率限制误触发**：内部请求不该消耗用户配额
4. **Memory/RAG 重复注入**：会污染模型输入

所以第 6 步的检测就是：**"这是 Looper 自己发出的内部请求吗？如果是，跳过普通插件处理，直接路由到目标模型。"**

检测依据是请求 header 中的 `x-vsr-looper-request: true`（见 `client.go:163`）。

---

## 详细例子：Confidence 算法（从小模型逐步升级到大模型）

### 场景设定

**配置**：
```yaml
decisions:
  - name: "coding_decision"
    algorithm:
      type: "confidence"
      confidence:
        threshold: 0.7          # 置信度阈值
        confidence_method: "avg_logprob"
        escalation_order: "size"  # 按模型大小升级
    model_refs:
      - model: "Qwen/Qwen2.5-3B-Instruct"    # 3B 小模型
      - model: "Qwen/Qwen2.5-14B-Instruct"   # 14B 中模型  
      - model: "Qwen/Qwen2.5-72B-Instruct"   # 72B 大模型
```

**用户请求**：
```json
{
  "model": "auto",
  "messages": [{"role": "user", "content": "实现一个支持并发安全的 LRU Cache，要求 O(1) 时间复杂度"}]
}
```

### 执行流程

#### 阶段 1：外部请求进入 ExtProc（正常 16 步）

```
用户 → Envoy → ExtProc.handleRequestBody()
```

- 步骤 1-5：解析请求，model = "auto"
- **步骤 6**：检查 header `x-vsr-looper-request` → **不存在** → 不是 Looper 内部请求，继续
- 步骤 7：提取 userContent = "实现一个支持并发安全的 LRU Cache..."
- 步骤 8：决策评估 → 匹配到 `coding_decision`，发现有 `algorithm` 配置
- 步骤 9-15：安全检查、RAG 等
- **步骤 16**：`shouldUseLooper(decision)` → **true**（有 algorithm + 多个 ModelRefs）
  - 调用 `handleLooperExecution()` → 创建 `ConfidenceLooper`

#### 阶段 2：Looper 执行（多轮调用）

**第 1 轮：尝试 3B 小模型**

Looper 通过 HTTP 发送请求到 Envoy（自己）：
```http
POST /v1/chat/completions HTTP/1.1
Content-Type: application/json
x-vsr-looper-request: true           ← 关键标识
x-vsr-looper-iteration: 1
x-vsr-looper-decision: coding_decision

{
  "model": "Qwen/Qwen2.5-3B-Instruct",
  "messages": [{"role": "user", "content": "实现一个支持并发安全的 LRU Cache..."}],
  "logprobs": true,
  "stream": false
}
```

这个请求再次进入 ExtProc：
```
Looper → Envoy → ExtProc.handleRequestBody()
```

- 步骤 1-5：解析请求
- **步骤 6**：检查 header `x-vsr-looper-request` → **"true"** → **是 Looper 内部请求！**
  - 调用 `handleLooperInternalRequestWithPlugins()`
  - 跳过鉴权、速率限制、Memory、RAG
  - 只做必要的 body 改写和 header 注入
  - 直接路由到 3B 模型的 vLLM 实例
  - **return**（不继续步骤 7-16）

3B 模型返回：
```json
{
  "choices": [{
    "message": {"content": "这是一个 LRU Cache 的实现...（代码比较粗糙）"},
    "logprobs": {"content": [{"logprob": -2.1}, {"logprob": -1.8}, ...]}
  }]
}
```

Looper 评估置信度：
```
average_logprob = -1.9 → 归一化后 confidence = 0.37
threshold = 0.7
0.37 < 0.7 → 不满足！→ 升级到更大模型
```

**第 2 轮：尝试 14B 中模型**

再次发送请求（同样带 `x-vsr-looper-request: true`）：
```http
POST /v1/chat/completions
x-vsr-looper-request: true
x-vsr-looper-iteration: 2
x-vsr-looper-decision: coding_decision

{"model": "Qwen/Qwen2.5-14B-Instruct", "messages": [...], "logprobs": true}
```

进入 ExtProc → 步骤 6 检测到 Looper 请求 → 短路 → 直接路由到 14B

14B 模型返回：
```
average_logprob = -0.6 → 归一化后 confidence = 0.80
0.80 ≥ 0.7 → 满足阈值！✓
```

#### 阶段 3：Looper 返回聚合结果

Looper 使用 14B 模型的响应作为最终结果，构建 `ImmediateResponse` 直接返回给 Envoy/用户：

```http
HTTP/1.1 200 OK
content-type: application/json
x-vsr-looper-model: Qwen/Qwen2.5-14B-Instruct
x-vsr-looper-models-used: Qwen/Qwen2.5-3B-Instruct,Qwen/Qwen2.5-14B-Instruct
x-vsr-looper-iterations: 2
x-vsr-looper-algorithm: confidence

{"choices": [{"message": {"content": "完整的并发安全 LRU Cache 实现..."}}]}
```

---

## 另一个例子：ReMoM 算法（多模型综合推理）

ReMoM 更复杂，它不是「升级」，而是「多个模型并行回答 + 综合」：

```
Round 1: 并行调用 4 个模型（如 Qwen-14B × 2, DeepSeek-7B × 2）
         每个调用都带 x-vsr-looper-request: true → 直接路由
         
Round 2: 把 Round 1 的 4 个回答作为 "Reference Responses"，
         构建综合 prompt，再调用 1 个模型生成最终答案
         
最终返回: Round 2 的综合答案
```

---

## 总结

| 方面 | 说明 |
|------|------|
| **Looper 的角色** | 多模型编排引擎，对用户透明 |
| **为什么要检测** | 防止递归、重复安全检查、重复扣费 |
| **检测方式** | Header `x-vsr-looper-request: true` |
| **检测后的处理** | 跳过大部分插件，直接路由到目标模型 |
| **支持的算法** | confidence（逐步升级）、ratings（评分选优）、remom（多模型综合）、rl_driven（强化学习驱动） |

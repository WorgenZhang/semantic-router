# ext_proc failure_mode_allow=true 与 vllm_dynamic_cluster 路由分析

## 问题

当 `envoy.filters.http.ext_proc` 的 `failure_mode_allow: true` 生效（即 ext_proc 服务运行失败）时，请求路由到 `vllm_dynamic_cluster` 会发生什么？整个链路是怎样的？会路由到什么模型？会使用默认模型吗？

---

## 一、正常链路回顾

正常情况下，完整的请求链路如下：

```
Client Request (port 8801)
    │
    ▼
┌─────────────────────────────────────────────────┐
│  Envoy Filter Chain                              │
│                                                  │
│  1. ext_authz (Authorino, port 50052)            │
│     ├─ 验证 Bearer Token                         │
│     └─ 注入 x-user-openai-key / x-user-anthropic-key │
│                                                  │
│  2. ext_proc (Semantic Router, port 50051)        │
│     ├─ Phase 1: Request Headers → CONTINUE       │
│     ├─ Phase 2: Request Body → 核心路由逻辑       │
│     │   ├─ 信号评估 (keywords, embeddings, etc.)  │
│     │   ├─ Decision Engine 匹配决策               │
│     │   ├─ 模型选择 (Elo/RouterDC/Static/ML等)    │
│     │   ├─ Endpoint 选择 (最优后端实例)            │
│     │   └─ 设置关键 Headers:                      │
│     │       • x-selected-model = "phi4-mini"      │
│     │       • x-vsr-destination-endpoint = "10.0.1.5:8000" │
│     │       • Authorization = "Bearer xxx"        │
│     │       • 修改 body 中的 model 字段            │
│     ├─ Phase 3: Response Headers → 追踪 headers   │
│     └─ Phase 4: Response Body → 缓存/指标         │
│                                                  │
│  3. Router Filter → 根据 headers 执行路由         │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│  Route Matching (基于 x-selected-model header)    │
│                                                  │
│  Route 1: prefix "claude-"                       │
│     → anthropic_api_cluster (api.anthropic.com)  │
│                                                  │
│  Route 2: regex "^(gpt-|o1-|o3-|chatgpt-).*"    │
│     → openai_api_cluster (api.openai.com)        │
│                                                  │
│  Route 3: Default (兜底路由)                      │
│     → vllm_dynamic_cluster                       │
│       (使用 x-vsr-destination-endpoint 确定后端)  │
└─────────────────────────────────────────────────┘
```

**正常情况下 ext_proc 的关键作用：**
- 设置 `x-selected-model` header → 决定 Envoy 将请求路由到哪个 cluster
- 设置 `x-vsr-destination-endpoint` header → 当路由到 `vllm_dynamic_cluster` 时，告诉 Envoy 具体的后端 IP:Port
- 修改请求 body 中的 `model` 字段 → 将模型别名替换为真实模型名
- 设置 `Authorization` header → 提供上游 API 认证凭证

---

## 二、ext_proc 失败时的链路分析

### 2.1 failure_mode_allow=true 的含义

```yaml
# config/envoy.yaml 第 94 行
failure_mode_allow: true
```

当 ext_proc 服务发生以下任何一种故障时：
- ext_proc gRPC 服务不可达（进程崩溃、网络断开）
- ext_proc gRPC 连接超时
- ext_proc 返回 gRPC 错误码
- ext_proc 处理超时（超过 `message_timeout: 300s`）

Envoy 的行为是：**跳过 ext_proc 处理，让请求继续通过 filter chain，就好像 ext_proc 从未存在过一样。**

### 2.2 失败时的实际链路

```
Client Request: POST /v1/chat/completions
  Body: { "model": "auto", "messages": [...] }
  Headers: Authorization: Bearer <user-token>
    │
    ▼
┌─────────────────────────────────────────────────┐
│  1. ext_authz (正常运行)                         │
│     ├─ 验证 Bearer Token ✅                      │
│     └─ 注入 x-user-openai-key 等 headers ✅      │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│  2. ext_proc ❌ 失败！                           │
│     ├─ failure_mode_allow=true → 跳过            │
│     └─ 请求 headers 和 body 保持原样，不做任何修改 │
│                                                  │
│     ❌ 不会设置 x-selected-model                  │
│     ❌ 不会设置 x-vsr-destination-endpoint        │
│     ❌ 不会修改 body 中的 model 字段              │
│     ❌ 不会设置上游 Authorization                 │
│     ❌ 不会执行任何信号评估/决策逻辑              │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│  3. Envoy Route Matching                         │
│                                                  │
│  检查 x-selected-model header:                   │
│    → header 不存在！(ext_proc 没有设置它)         │
│                                                  │
│  Route 1: x-selected-model prefix "claude-"      │
│    → ❌ 不匹配 (header 不存在)                    │
│                                                  │
│  Route 2: x-selected-model regex "^(gpt-|o1-|…)" │
│    → ❌ 不匹配 (header 不存在)                    │
│                                                  │
│  Route 3: Default (prefix "/")                   │
│    → ✅ 匹配! → vllm_dynamic_cluster             │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│  4. vllm_dynamic_cluster 尝试转发                │
│                                                  │
│  cluster 配置:                                   │
│    type: ORIGINAL_DST                            │
│    lb_policy: CLUSTER_PROVIDED                   │
│    original_dst_lb_config:                       │
│      use_http_header: true                       │
│      http_header_name: "x-vsr-destination-endpoint" │
│                                                  │
│  检查 x-vsr-destination-endpoint header:          │
│    → ❌ header 不存在！(ext_proc 没有设置它)      │
│                                                  │
│  结果: Envoy 无法确定上游目标地址                 │
│    → 返回 503 Service Unavailable                │
│    → response_flags 可能包含 "UH" (No healthy    │
│      upstream) 或 "NC" (No cluster found)        │
└─────────────────────────────────────────────────┘
    │
    ▼
  Client 收到 503 错误
```

### 2.3 关键结论

| 项目 | 结果 |
|------|------|
| **会路由到哪个 cluster？** | `vllm_dynamic_cluster`（默认兜底路由） |
| **会路由到什么模型？** | **不会路由到任何模型** |
| **会使用默认模型吗？** | **不会。** 默认模型逻辑 (`r.Config.DefaultModel`) 存在于 ext_proc 代码中，ext_proc 失败意味着这段代码根本不会执行 |
| **最终结果** | **503 Service Unavailable** |

---

## 三、为什么会返回 503？详细技术解释

### 3.1 ORIGINAL_DST cluster 的工作原理

`vllm_dynamic_cluster` 的配置（`config/envoy.yaml` 第 135-147 行）：

```yaml
- name: vllm_dynamic_cluster
  type: ORIGINAL_DST          # 使用原始目的地
  lb_policy: CLUSTER_PROVIDED  # 由 cluster 自身决定负载均衡
  original_dst_lb_config:
    use_http_header: true                        # 从 HTTP header 获取目标地址
    http_header_name: "x-vsr-destination-endpoint"  # 指定 header 名称
```

这个 cluster **没有预定义的 endpoints**。它完全依赖 `x-vsr-destination-endpoint` header 来确定目标后端地址。

### 3.2 缺失 header 时的行为

当 `x-vsr-destination-endpoint` header 不存在时：

1. `ORIGINAL_DST` cluster 的 LB 策略尝试从 header 获取目标地址
2. Header 不存在 → 没有可用的目标地址
3. 没有 fallback（`ORIGINAL_DST` 没有静态 endpoints 列表作为后备）
4. Envoy 返回 **503 Service Unavailable**，response flag 为 `UH`（No healthy upstream）

### 3.3 即使有 header 也会失败的情况

即使客户端自己设置了 `x-vsr-destination-endpoint`，还有另一个问题：

- 请求 body 中的 `model` 字段仍然是 `"auto"` 或用户指定的别名（如 `"phi4-mini"`）
- vLLM 后端不认识 `"auto"` 这个模型名
- vLLM 后端可能也不认识别名，需要的是完整模型名如 `"microsoft/phi-4-mini-instruct"`
- 结果：vLLM 返回 **400 Bad Request** 或 **404 Model Not Found**

---

## 四、可能触发 ext_proc 失败的场景

| 场景 | 说明 |
|------|------|
| Semantic Router 进程崩溃 | Go 程序 panic/OOM/被 kill |
| Semantic Router 启动中 | 服务还未 ready，gRPC 端口未监听 |
| 网络分区 | Envoy 与 127.0.0.1:50051 之间网络异常 |
| gRPC 连接池耗尽 | 并发过高，所有连接都在使用中 |
| 处理超时 | 某个请求处理超过 `message_timeout: 300s` |
| gRPC 错误 | Semantic Router 内部代码抛出未捕获的 gRPC 错误 |

---

## 五、与 ext_authz 的对比

注意 `ext_authz` 和 `ext_proc` 的 `failure_mode_allow` 设置截然不同：

```yaml
# ext_authz: failure_mode_allow = false (第 79 行)
# → 如果 Authorino 失败，请求立即被拒绝 (403)
# → 安全优先：宁可拒绝服务，也不放行未认证请求

# ext_proc: failure_mode_allow = true (第 94 行)
# → 如果 Semantic Router 失败，请求继续通过
# → 可用性优先：但实际上因为缺少关键 headers，请求最终也会失败 (503)
```

**设计意图对比：**

| Filter | failure_mode_allow | 设计理由 |
|--------|-------------------|----------|
| ext_authz | `false` | 安全性：未认证请求绝不能通过 |
| ext_proc | `true` | 可用性：理论上希望在路由服务故障时仍能服务请求，但实际效果有限 |

---

## 六、failure_mode_allow=true 在当前架构中的实际意义

### 6.1 当前状态：形同虚设

在当前的架构下，`failure_mode_allow: true` 的"容错"效果非常有限：

1. **没有真正的 fallback**：`vllm_dynamic_cluster` 是 `ORIGINAL_DST` 类型，没有静态后端列表，没有 `x-vsr-destination-endpoint` 就无法工作
2. **没有默认模型注入**：默认模型逻辑在 ext_proc 代码内部，ext_proc 失败意味着默认模型逻辑也不会执行
3. **最终结果依然是错误**：503 而不是 500，但对用户来说没有本质区别

### 6.2 这个配置的唯一好处

`failure_mode_allow: true` 的实际好处是：

- **Envoy 本身不会崩溃或返回 500**：它会优雅地跳过 ext_proc，而不是因为 filter 错误直接断开连接
- **Access Log 正常记录**：请求会完整走完 filter chain，access log 能记录到 503 和相关的 response flags，有利于排查问题
- **不会阻塞其他已有正确 header 的请求**：如果某个客户端在请求中已经手动设置了 `x-selected-model` 和 `x-vsr-destination-endpoint`，理论上可以绕过 ext_proc 直接路由成功（但这在正常使用中不会发生）

---

## 七、如果要实现真正的容错，可能的改进方案

### 方案 1: 为 vllm_dynamic_cluster 添加静态 fallback endpoints

```yaml
- name: vllm_dynamic_cluster
  type: STATIC                    # 改为 STATIC
  lb_policy: ROUND_ROBIN          # 改为轮询
  load_assignment:
    cluster_name: vllm_dynamic_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: 10.0.1.5   # 默认 vLLM 实例
              port_value: 8000
```

**问题**：失去了动态路由的灵活性，且 body 中的 `model` 字段仍未修改。

### 方案 2: 添加一个独立的 fallback cluster

```yaml
routes:
  # ... 现有路由 ...

  # Fallback: 当没有 x-selected-model 时
  - match:
      prefix: "/"
    route:
      cluster: vllm_fallback_cluster  # 指向一个有静态 endpoint 的 cluster
      timeout: 300s
```

### 方案 3: 将 failure_mode_allow 改为 false

```yaml
failure_mode_allow: false
```

这样 ext_proc 失败时，Envoy 会直接返回 **500 Internal Server Error**，语义上更清晰——告诉客户端"路由服务不可用"，而不是让请求走到 503 才失败。

### 方案 4: 在 ext_proc 内部增强容错

在 Semantic Router 代码中增加更健壮的错误恢复：
- 确保即使所有信号评估失败，也能 fallback 到 `DefaultModel`
- 确保即使 endpoint 选择失败，也能返回一个可用的默认 endpoint
- 这是 ext_proc **内部**的容错，不是 ext_proc **整体失败**时的容错

---

## 八、总结

```
ext_proc 失败 (failure_mode_allow=true)
    │
    ▼
请求跳过 ext_proc，headers 和 body 保持原样
    │
    ▼
x-selected-model header 不存在
    │
    ▼
Envoy 路由匹配: Route 1 ❌ Route 2 ❌ Route 3 ✅ (默认兜底)
    │
    ▼
进入 vllm_dynamic_cluster (ORIGINAL_DST)
    │
    ▼
x-vsr-destination-endpoint header 不存在
    │
    ▼
ORIGINAL_DST 无法确定后端地址
    │
    ▼
╔═══════════════════════════════════════╗
║  最终结果: 503 Service Unavailable    ║
║  不会路由到任何模型                    ║
║  不会使用任何默认模型                  ║
║  默认模型逻辑在 ext_proc 代码内部，    ║
║  ext_proc 整体失败时该逻辑不会执行     ║
╚═══════════════════════════════════════╝
```

**核心结论**：`failure_mode_allow: true` 在当前架构下不能提供真正的请求容错。ext_proc 失败时，请求最终会以 **503** 失败，因为 `vllm_dynamic_cluster` 依赖 ext_proc 设置的 `x-vsr-destination-endpoint` header 来确定后端地址，而这个 header 在 ext_proc 失败时不会被设置。
